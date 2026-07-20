---
permalink: /docs/charts/
title: 'Charts'
toc: true
---

## Overview

The `floorplan.chart_set` service renders live charts directly inside the floorplan SVG. When the service is called, the SVG element that is targeted by the rule is replaced by a chart of the same size and position. The chart is redrawn whenever the state of one of its entities changes, so the floorplan always shows current data.

Three kinds of chart are provided. The `history-graph` type and the `gauge` type replicate the built in Home Assistant history graph card and gauge card. The `apex-chart` type accepts a complete [ApexCharts](https://apexcharts.com/docs/options/) configuration and can be used to build fully custom charts.

Charts are rendered as static SVG content. They are scaled and panned together with the rest of the floorplan. Tooltips, toolbars and animations are not available, because the chart is converted to a plain image each time it is drawn.

### How a chart is placed

A placeholder element is added to the floorplan SVG at the location where the chart should appear. A simple rectangle is the easiest choice. The rectangle is given an id and that id is used as the target element of the chart rule. When the rule fires, the rectangle is replaced by the chart.

The bounding box of the placeholder defines the size and the position of the chart, so the rectangle should be drawn at the size that the chart is expected to fill. The ideal size and aspect ratio depend on the chart type and its content, so some trial and error in the SVG editor is usually the quickest way to get a good result. Legends, axis labels and gauge text all take up part of the available space, which is worth keeping in mind when a chart looks cramped.

```html
<rect id="my_chart" x="340" y="60" width="420" height="300" />
```

The entities listed in the service data are registered as triggers for the rule, which means the chart is refreshed whenever one of those entities changes state. A chart whose entities update rarely is redrawn rarely. Charts that are rendered while their floorplan page is hidden may be drawn incorrectly, and they are corrected on the next state change after the page becomes visible.

```yaml
rules:
  - element: my_chart
    state_action:
      - service: floorplan.chart_set
        service_data:
          type: history-graph
          entities:
            - sensor.temperature
```

Other rule options can be combined with a chart rule. For example, `tap_action: more-info` can be used to open the more info dialog when the chart is tapped.

## History graph

The `history-graph` type replicates the Home Assistant history graph card. The state history of the listed entities is fetched from Home Assistant and drawn as a stepped line chart. The line colors are taken from the active Home Assistant theme and match the colors that are used by the built in history graph card.

The following service data options are supported. If an option is not listed in this table then it is not supported.

| Option               | Required | Default | Description                                                                                          |
| -------------------- | -------- | ------- | ---------------------------------------------------------------------------------------------------- |
| `type`               | yes      |         | Must be `history-graph`. The value `statistics-graph` is also accepted and behaves identically.       |
| `entities`           | yes      |         | A list of entity ids. An entry can also be an object with `entity` and `name` keys, where `name` overrides the label that is shown in the legend. |
| `hours_to_show`      | no       | `24`    | The number of hours of history that is shown.                                                        |
| `refresh_interval`   | no       | `10`    | The minimum number of seconds between history fetches.                                               |
| `title`              | no       |         | A title that is drawn above the chart.                                                               |
| `show_names`         | no       | `true`  | Whether the legend is shown.                                                                         |
| `theme`              | no       | `light` | The ApexCharts theme mode. The accepted values are `light` and `dark`.                               |
| `background`         | no       |         | A CSS color that is painted behind the chart.                                                        |
| `apex_chart_options` | no       |         | Additional ApexCharts options that are applied over the generated chart. The merge behaviour is described below the table. |

Any option from the [ApexCharts options reference](https://apexcharts.com/docs/options/) can be entered under `apex_chart_options`. Each top level key replaces the corresponding generated key completely, so when a key such as `legend` or `stroke` is provided, the whole generated value for that key is replaced rather than merged into it. The `series` key is protected and is always generated from the entity history. The settings that are enforced by the floorplan (the chart size, tooltips, the toolbar and animations, as described in the custom charts section) are applied after the merge and cannot be changed here.

A simple history graph is configured as follows.

```yaml
rules:
  - element: graph.placeholder
    tap_action: more-info
    state_action:
      - service: floorplan.chart_set
        service_data:
          type: history-graph
          entities:
            - entity: sensor.speedtest_download
            - entity: sensor.speedtest_upload
          hours_to_show: 24
          show_names: true
```

The appearance of the chart can be changed through `apex_chart_options`. In the following example the chart is drawn on a black background with a dark theme and a smooth line style.

```yaml
rules:
  - element: graph1
    tap_action: more-info
    state_action:
      - service: floorplan.chart_set
        service_data:
          type: history-graph
          entities:
            - entity: sensor.wupws_temp
            - entity: sensor.living_room_aircon
          hours_to_show: 24
          show_names: false
          theme: dark
          apex_chart_options: |
            >
            return {
              chart: {
                type: 'area',
                background: '#000000',
              },
              stroke: {
                curve: 'smooth',
                width: 3,
              },
            };
```

The following limitations apply to the history graph.

- The `statistics-graph` type is accepted for compatibility but it renders the same state history as `history-graph`. Long term statistics are not fetched.
- When the listed entities have different units of measurement, only one unit group is drawn. Entities that share a unit should be charted together.
- Entities with non numeric states are not drawn.

## Gauge

The `gauge` type replicates the Home Assistant gauge card. The gauge is drawn as native SVG and supports the same severity and segment options as the built in card.

The following service data options are supported. If an option is not listed in this table then it is not supported.

| Option       | Required | Default | Description                                                                                             |
| ------------ | -------- | ------- | ------------------------------------------------------------------------------------------------------- |
| `type`       | yes      |         | Must be `gauge`.                                                                                        |
| `entity`     | yes      |         | The entity whose state is shown. The entity is also registered as the trigger for the rule.             |
| `name`       | no       |         | The name that is shown below the value. The friendly name of the entity is used when no name is given.  |
| `unit`       | no       |         | The unit that is shown after the value. The unit of measurement of the entity is used when no unit is given. |
| `min`        | no       | `0`     | The minimum value of the gauge.                                                                         |
| `max`        | no       | `100`   | The maximum value of the gauge.                                                                         |
| `needle`     | no       | `false` | Whether the gauge is drawn with a needle. Severity and segment colors are drawn as a fixed scale when the needle is enabled. |
| `severity`   | no       |         | An object with `green`, `yellow` and `red` keys. Each key defines the value at which that color starts. |
| `segments`   | no       |         | A list of objects with `from`, `color` and optional `label` keys. When a segment defines a label, the label is shown in place of the numeric value. |
| `value`      | no       |         | An expression that overrides the displayed value. The state of the entity is used when no value is given. |
| `background` | no       |         | A CSS color that is painted behind the gauge.                                                           |
| `theme`      | no       |         | The name of a Home Assistant theme. The theme variables of that theme are applied to the gauge.         |

A simple gauge is configured as follows.

```yaml
rules:
  - element: my_gauge
    state_action:
      - service: floorplan.chart_set
        service_data:
          type: gauge
          entity: sensor.humidity
          min: 0
          max: 100
          severity:
            green: 0
            yellow: 60
            red: 80
```

In the following example the gauge is drawn with a needle and a dark background.

```yaml
rules:
  - element: graph2
    state_action:
      - service: floorplan.chart_set
        service_data:
          type: gauge
          name: Temperature
          background: '#1C1C1C'
          entity: input_number.temp_graph_test
          needle: true
          severity:
            green: 15
            yellow: 25
            red: 30
          max: 40
```

The gauge is styled with the theme variables of the active Home Assistant theme. The text and needle colors follow `--primary-text-color` and the unfilled part of the arc follows `--primary-background-color`. On floorplans where those colors do not fit the background image, the gauge can be restyled from the floorplan stylesheet, because the gauge is plain SVG inside the floorplan. The available classes are `dial`, `value`, `needle`, `level`, `value-text` and `name-text`.

```css
#my_gauge .value-text,
#my_gauge .name-text {
  fill: #212121;
}

#my_gauge .dial {
  stroke: #e8e8e8;
}
```

## Custom charts

The `apex-chart` type renders a chart from a raw ApexCharts configuration. This type is intended for charts that go beyond the history graph and the gauge, for example radial bars, horizontal bars and charts whose series are computed from several entities.

The following service data options are supported. If an option is not listed in this table then it is not supported. In particular, the `title`, `theme`, `background`, `hours_to_show` and `show_names` options of the history graph are not applied to this chart type. A background is set with `chart.background` inside `apex_chart_options` instead.

| Option               | Required | Default | Description                                                                                            |
| -------------------- | -------- | ------- | ------------------------------------------------------------------------------------------------------ |
| `type`               | yes      |         | Must be `apex-chart`.                                                                                  |
| `entities`           | no       |         | A list of entity ids that trigger a redraw of the chart. Entities that are only referenced inside the template are not detected automatically, so they should be listed here or on the rule itself. |
| `refresh_interval`   | no       | `10`    | The minimum number of seconds between history fetches that are made through the `getStateHistory` helper. |
| `apex_chart_options` | yes      |         | The ApexCharts configuration. The value is usually a code block that returns the configuration object. |

The `apex_chart_options` value is passed directly to the ApexCharts library. The available options are documented in the [ApexCharts options reference](https://apexcharts.com/docs/options/). Note that this is not the configuration format of the apexcharts card that is distributed through HACS. Card level options from that project are not supported here.

The following chart types have been verified to render on the floorplan. The types are `line`, `area`, `scatter`, `bubble`, `rangeArea`, `bar`, `rangeBar`, `candlestick`, `boxPlot`, `violin`, `pie`, `donut`, `polarArea`, `radialBar`, `radar`, `heatmap` and `treemap`. Vertical columns are drawn with the `bar` type, which draws vertical bars unless `plotOptions.bar.horizontal` is enabled.

ApexCharts options are passed through without filtering, but the floorplan converts every chart to static SVG, so options only take effect when their result is part of the drawn image. The behaviour can be summarised as follows.

- Options that shape the drawn image are supported. This covers the series, colors, fills, strokes, grids, axes, data labels, plot options, legends and annotations.
- The width and height of the chart are taken from the bounding box of the target element. The `chart.width` and `chart.height` options are ignored.
- Tooltips, the toolbar and animations are always disabled and the related options are ignored.
- Options that depend on a live chart have no effect. This covers zooming, panning, selection, event callbacks, keyboard navigation, drilldown and the download menu.
- Responsive breakpoints are based on the browser window rather than the placeholder element, so they are not useful on a floorplan.

A simple custom chart is configured as follows. The series is computed from the current state of an entity.

```yaml
rules:
  - element: cpu_gauge
    state_action:
      - service: floorplan.chart_set
        service_data:
          type: apex-chart
          entities:
            - sensor.processor_use
          apex_chart_options: |
            >
            return {
              chart: { type: 'radialBar' },
              series: [parseFloat(entities['sensor.processor_use'].state)],
              labels: ['CPU'],
            };
```

In the following example a horizontal bar chart is built from several entities. All of the entities are listed under `entities`, so the chart is redrawn when any of them changes.

```yaml
rules:
  - element: graph3
    tap_action: more-info
    state_action:
      - service: floorplan.chart_set
        service_data:
          type: apex-chart
          entities:
            - sensor.processor_use
            - sensor.memory_use_percent
          apex_chart_options: |
            >
            return {
              chart: {
                type: 'bar',
                background: '#000000',
              },
              plotOptions: {
                bar: {
                  horizontal: true,
                  barHeight: '10%',
                },
              },
              dataLabels: {
                enabled: false,
              },
              colors: ['#FF0000', '#546E7A'],
              series: [{
                data: [
                  { x: 'CPU %', y: parseFloat(entities['sensor.processor_use'].state) },
                  { x: 'Mem %', y: parseFloat(entities['sensor.memory_use_percent'].state) },
                ],
              }],
              grid: {
                show: false,
              },
              xaxis: {
                max: 100,
              },
            };
```

In the following example three entities are drawn as concentric radial bars with a legend.

```yaml
rules:
  - element: graph4
    tap_action: more-info
    state_action:
      - service: floorplan.chart_set
        service_data:
          type: apex-chart
          entities:
            - sensor.processor_use
            - sensor.memory_use_percent
            - sensor.disk_use_percent
          apex_chart_options: |
            >
            return {
              series: [
                parseFloat(entities['sensor.processor_use'].state),
                parseFloat(entities['sensor.memory_use_percent'].state),
                parseFloat(entities['sensor.disk_use_percent'].state)
              ],
              chart: {
                type: 'radialBar',
              },
              labels: ['CPU', 'Mem', 'Disk'],
              plotOptions: {
                radialBar: {
                  startAngle: 0,
                  endAngle: 359,
                  hollow: {
                    margin: 10,
                    size: "30%",
                    background: "#000000",
                  },
                  track: {
                    background: "#202020",
                  },
                },
              },
              legend: {
                show: true,
                position: 'top',
                labels: {
                  useSeriesColors: true,
                },
                formatter: function (val) {
                  return val + '%';
                },
              },
            };
```

### Fetching state history in a template

The `getStateHistory` helper is available inside the templates of a `chart_set` action. It fetches the raw state history of one or more entities, so a fully custom chart can show history instead of only current states. The helper is not available in the templates of other floorplan services.

The helper is called with a comma separated string of entity ids and it must be awaited. The result is an object that is keyed by entity id. Each entry is a list of raw history states with short keys. The `s` key holds the state. The `lu` key holds the last updated time in epoch seconds. The `lc` key holds the last changed time and it is only present when it differs from `lu`. The `a` key holds the attributes and it is only present on the first entry.

The fetch window is fixed at the last 24 hours. Calls are throttled to one call per `refresh_interval` seconds.

```yaml
rules:
  - element: my_custom_chart
    entity: sensor.power
    state_action:
      - service: floorplan.chart_set
        service_data:
          type: apex-chart
          refresh_interval: 30
          apex_chart_options: |
            >
            const history = await getStateHistory('sensor.power');
            const series = history['sensor.power'].map((state) => ({
              x: (state.lc ?? state.lu) * 1000,
              y: parseFloat(state.s),
            }));
            return {
              chart: { type: 'line' },
              series: [{ name: 'Power', data: series }],
              xaxis: { type: 'datetime' },
            };
```

