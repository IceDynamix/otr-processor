# otr-processor

<details>
<summary>original readme</summary>

Rating calculation engine for the o!TR platform. Processes osu! tournament
data to calculate player skill ratings using a PlackettLuce model.

Please see the
[Development Guide](https://docs.otr.stagec.net/Development/Development-Guide) for more information.

</details>

## map ratings

map ratings are individual per mod combination and per mode (if you ignore converts this doesn't matter). a map gets an initial rating depending on star rating, however the public otr replica doesn't ship with star rating for mods, so i use [this very crude heuristic](https://github.com/IceDynamix/otr-processor/blob/map-ratings/src/model/rating_utils.rs#L116) instead.

that heuristic takes the star rating and converts it into a rating. i used some linear interpolation stuff in order to figure out a good approximation, mostly based on the dataset of the 50th percentile of star rating vs player rating (see [this](https://github.com/IceDynamix/otr-processor/blob/map-ratings/src/model/rating_utils.rs#L136))

i chose a default volatility of 200, compared to the player's 400. i also clamped the initial rating to [100, 2500], because some maps have broken star ratings (e.g. aspire).

while processing a match, a map is treated like a separate player with a [specific score that depends on the ruleset that i chose based on recorded no-mod scores in the otr dataset](https://github.com/IceDynamix/otr-processor/blob/map-ratings/src/model/otr_model.rs#L268)

the map is [rated against the other players in the game](https://github.com/IceDynamix/otr-processor/blob/map-ratings/src/model/otr_model.rs#L206) and the result is stored in the new table `beatmap_ratings`. changes to the beatmap ratings are also tracked and stored in `beatmap_rating_adjustments`. these tables don't exist in the public replica, the tables are (re)created in the processor (which does not follow the migration format that otr uses, afaik its drizzle orm). you can find the exact sql [here](https://github.com/IceDynamix/otr-processor/blob/map-ratings/src/database/db.rs#L129) 

## compiling results yourself

most instructions are the same as the [original development guide](https://docs.otr.stagec.net/Development/Development-Guide#prerequisites)

to compile map ratings, follow these steps

- install prerequisites
  - git
  - [docker](https://www.docker.com/)
  - [rust](https://rust-lang.org/tools/install/)
  - [public database replica](https://data.otr.stagec.net)
  - (installing bun is _not_ required unless you want to set up the web interface)

```sh
git clone https://github.com/osu-tournament-rating/otr-web.git
cd otr-web
cp .env.example .env

# start the database
docker compose up -d db rabbitmq

# import the database replica. takes a while, keep it running even if it seems to be stuck
gunzip -c /path/to/replica.gz | docker exec -i db bash -c "psql -U postgres -d template1 -c 'DROP DATABASE IF EXISTS postgres;' && psql -U postgres -d template1 -c 'CREATE DATABASE postgres;' && psql -U postgres -d postgres"

cd ..
git clone https://github.com/IceDynamix/otr-processor.git # (note that this is my repo)
cd otr-processor
git checkout map-ratings

# environment variables, could use .env but its fine for now
export CONNECTION_STRING="postgresql://postgres:password@localhost:5432/postgres" 
export IGNORE_CONSTRAINTS=true
cargo run
```

process takes about 15 minutes on my pc

to export beatmap ratings to a csv, use following command

```
docker exec -it db psql -U postgres -d postgres -c "\
COPY (
    SELECT b.osu_id,
           b.ruleset,
           br.mods,
           s.artist || ' - ' || s.title || ' [' || b.diff_name || ']'
                            AS "map",
           b.sr,
           br.rating        AS "current_rating",
           bra.rating_after AS "initial_rating",
           br.volatility    AS "current_volatility"
    FROM public.beatmap_ratings br
             JOIN public.beatmaps b ON br.beatmap_id = b.id
             JOIN public.beatmapsets s ON s.id = b.beatmapset_id
             JOIN public.beatmap_rating_adjustments bra ON br.id = bra.beatmap_rating_id AND bra.adjustment_type = 0
    ORDER BY br.rating DESC
) TO STDOUT WITH CSV HEADER;" > ratings.csv
```

optionally, add `WHERE br.ruleset = 0` to filter by mode

- 0=osu!
- 1=osu!taiko
- 2=osu!catch
- 3=osu!mania (Other) [No ratings are generated for this ruleset]
- 4=osu!mania 4K
- 5=osu!mania 7K
