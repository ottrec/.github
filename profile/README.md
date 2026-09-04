#### Website

- [**Today**](https://ottrec.ca/today) - upcoming activities
- [**Map**](https://ottrec.ca/map) - filterable map of facilities
- [**Schedules**](https://ottrec.ca/schedules) - view all schedules on one page
- [**Activities**](https://ottrec.ca/activities) - activity availability by area, facility, and time of day
- [**Advanced Search**](https://ottrec.ca/about/ottrecql) - create a custom set of schedules
- [**About**](https://ottrec.ca/about) - more info and write-ups about how stuff works

#### Datasets

<table>
    <thead>
        <tr>
            <th align="left">Format</th>
            <th align="left">Download</th>
            <th align="left">Schema</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>JSON (simplified)</td>
            <td><a href="https://data.ottrec.ca/export/latest.json">latest.json</a></td>
            <td><a href="https://data.ottrec.ca/export/schema.json">schema.json</a></td>
        </tr>
        <tr>
            <td>CSV (simplified)</td>
            <td><a href="https://data.ottrec.ca/export/latest.csv.zip">latest.csv.zip</a></td>
            <td><a href="https://data.ottrec.ca/export/schema.csv">schema.csv</a></td>
        </tr>
        <tr>
            <td>Protobuf (raw)</td>
            <td><a href="https://data.ottrec.ca/v1/latest/pb">latest.pb</a></td>
            <td><a href="https://data.ottrec.ca/v1/latest/proto">schema.proto</a></td>
        </tr>
    </tbody>
</table>

See <a href="https://data.ottrec.ca">data.ottrec.ca</a> for more information.

#### LLM Usage

The scraper and schema are entirely hand-written and verified. The dataset indexing, heuristics, and query language for the website are as well.

I read the diff of all dataset updates. I also have claude read them as well to catch the less-obvious things (using some tooling I gave it ideas for). I report typos and incorrect schedules to the city daily when I notice them (usually ~5 per week).

The website is designed by me at a high-level (features, layout, and architecture), but I delegate most of the implementation to Claude since I don't really enjoy front-end development. All long-form prose (and most of the shorter stuff) is written by me.

The data enrichment is fully maintained by Claude, but I review the code and make the decisions about how to interpret stuff.
