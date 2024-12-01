---
title: USA Chronic Indicators Map
toc: false
sql:
  disease_indicators: ./data/disease_indicators.csv
---

```js
const db = await DuckDBClient.of({ usa_disease_indicators: FileAttachment("./data/disease_indicators.csv") });
```

```js
const usaData = FileAttachment("./data/us-counties-10m.json").json();
```

```js
const queryParamsPerIndicator = [
    {
        indicatorName: "Chronic liver disease mortality",
        indicatorId: "ALC6_0",
        dataValueTypeId: "CRDRATE",
        mostRecentStartYear: 2020,
        mostRecentEndYear: 2020,
    },
    {
        indicatorName: "Asthma mortality rate",
        indicatorId: "AST4_1",
        dataValueTypeId: "CRDRATE",
        mostRecentStartYear: 2020,
        mostRecentEndYear: 2020,
    },
    {
        indicatorName: "Cancer of the prostate, mortality",
        indicatorId: "CAN11_2",
        dataValueTypeId: "AVGANNCRDRATE",
        mostRecentStartYear: 2011,
        mostRecentEndYear: 2015,
    },
    {
        indicatorName: "High cholesterol prevalence among adults aged greater than or equal to 18 years",
        indicatorId: "CVD5_0",
        dataValueTypeId: "CRDPREV",
        mostRecentStartYear: 2021,
        mostRecentEndYear: 2021,
    },
    {
        indicatorName: "Obesity among adults aged greater than or equal to 18 years",
        indicatorId: "NPAW1_1",
        dataValueTypeId: "CRDPREV",
        mostRecentStartYear: 2021,
        mostRecentEndYear: 2021,
    },
];
```

```js
const indicatorOptions = new Map(queryParamsPerIndicator.map(ind => [ind.indicatorName, ind.indicatorId]));

const indicatorId = view(Inputs.select(
    indicatorOptions,
    {
        label: "Indicator",
        value: "AST4_1",
    }
));
```

```js
const queryParams = queryParamsPerIndicator.find(metadata => metadata.indicatorId === indicatorId);

const asthma_mortality_rates_sql = db.sql`
SELECT *
FROM usa_disease_indicators
WHERE "YearEnd" = ${queryParams.mostRecentEndYear} AND
    "StratificationID1" = 'OVR' AND
    "QuestionID" = ${queryParams.indicatorId} AND
    "DataValueTypeID" = ${queryParams.dataValueTypeId} AND
    "GeoLocation" IS NOT NULL
`;
```

```js
import * as topojson from "npm:topojson-client";

const usStates = topojson.feature(usaData, usaData.objects.states);

const rows = [...asthma_mortality_rates_sql];

const stateMap = new Map();

rows.forEach(r => {
    const locationId = Number.parseInt(r["LocationID"]);

    if (r["DataValue"]) {
        const value = Number.parseFloat(r["DataValue"]);
        const scale = r["DataValueType"];

        stateMap.set(locationId, { value, scale });
    } else {
        stateMap.set(locationId, { value: 0, scale: "NA" });
    }
});

usStates.features.forEach(f => {
    const id = Number.parseInt(f.id);
    const innerData = stateMap.get(id);

    if (innerData) {
        f.properties.value = innerData.value;
        f.properties.scale = innerData.scale;
    } else {
        f.properties.value = 0;
        f.properties.scale = "NA";
    }
});
```

```js
Plot.plot({
    height: 800,
    width: 1000,
    projection: "albers-usa",
    color: {
        type: "linear",
        n: 8,
        scheme: "blues",
        label: `${queryParamsPerIndicator.find(ind => ind.indicatorId == indicatorId).indicatorName} (%) from ${queryParamsPerIndicator.find(ind => ind.indicatorId == indicatorId).mostRecentStartYear} to ${queryParamsPerIndicator.find(ind => ind.indicatorId == indicatorId).mostRecentEndYear}`,
        legend: true,
    },
    marks: [
        Plot.geo(usStates, {
            fill: "value",
            title: (d) => `${d.properties.name}: ${d.properties.scale} of ${d.properties.value}%`,
            tip: true
        })
    ]
})
```