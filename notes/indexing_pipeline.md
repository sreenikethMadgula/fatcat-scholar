
## Prototype Pipeline

Requires staff credentials in environment for `internetarchive` python library.

TODO: pass these credentials via ansible/dotenv

Generate complete SIM issue database:

    ia search "collection:periodicals collection:sim_microfilm mediatype:collection" --itemlist | rg "^pub_" > data/sim_collections.tsv
    ia search "collection:periodicals collection:sim_microfilm mediatype:texts" --itemlist | rg "^sim_" > data/sim_items.tsv

    cat data/sim_collections.tsv | parallel -j4 ia metadata {} | jq . -c | pv -l > data/sim_collections.json
    cat data/sim_items.tsv | parallel -j8 ia metadata {} | jq . -c | pv -l > data/sim_items.json

    python -m fatcat_scholar.issue_db init_db
    cat data/sim_collections.json | pv -l | python -m fatcat_scholar.issue_db load_pubs
    cat data/sim_items.json | pv -l | python -m fatcat_scholar.issue_db load_issues
    python -m fatcat_scholar.issue_db load_counts

Create QA elasticsearch index (localhost):

    http put ":9200/qa_scholar_fulltext_v01?include_type_name=true" < schema/scholar_fulltext.v01.json
    http put ":9200/qa_scholar_fulltext_v01/_alias/qa_scholar_fulltext"

Fetch "heavy" fulltext documents (JSON) for full SIM database:

    python -m fatcat_scholar.sim_pipeline run_issue_db | pv -l | gzip > data/sim_intermediate.json.gz

Re-use existing COVID-19 database to index releases:

    cat /srv/fatcat_covid19/metadata/2020-06-24/fatcat_hits.enrich.json \
        | jq -c .fatcat_release \
        | rg -v "^null" \
        | parallel -j8 --linebuffer --round-robin --pipe python -m fatcat_scholar.work_pipeline run_releases \
        | pv -l \
        | gzip > data/work_intermediate.json.gz

    => 48.3k 0:17:58 [44.8 /s]

Transform and index both into local elasticsearch:

	zcat data/work_intermediate.json.gz data/sim_intermediate.json.gz \
        | parallel -j8 --linebuffer --round-robin --pipe python -m fatcat_scholar.transform run_transform \
        | esbulk -verbose -size 100 -id key -w 4 -index qa_scholar_fulltext_v01 -type _doc

    => 132635 docs in 2m18.787824205s at 955.667 docs/s with 4 workers

