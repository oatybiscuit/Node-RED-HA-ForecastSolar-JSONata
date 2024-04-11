# Node-RED-HA-ForecastSolar-JSONata

Read Forecast.Solar **API** using Node-RED and JSONata for Home Assistant

[![GitHub Release](https://img.shields.io/github/v/release/oatybiscuit/Node-RED-HA-ForecastSolar-JSONata)](https://github.com/oatybiscuit/Node-RED-HA-ForecastSolar-JSONata/releases)
![GitHub Downloads (all assets, latest release)](https://img.shields.io/github/downloads/oatybiscuit/Node-RED-HA-ForecastSolar-JSONata/latest/total)
[![License: MIT](https://img.shields.io/github/license/oatybiscuit/Node-RED-HA-ForecastSolar-JSONata)](/LICENSE)

[![Node RED](https://img.shields.io/badge/Node--RED-8f0000)](https://nodered.org/)
[![JSONata](https://img.shields.io/badge/JSONata-285154)](https://jsonata.org/)
[![Forecast Solar](https://img.shields.io/badge/ForecastSolar-f050f8)](https://forecast.solar/)
[![Home Assistant](https://img.shields.io/badge/Home_Assistant-038fc7)](https://www.home-assistant.io/)

A **Node-RED** flow to read, analyze and save into context variables **Forecast Solar API forecast** using **JSONata**. Manage multiple solar arrays and store forecast with actual production history for yesterday, today, and tomorrow. Ability to create sensors in **Home Assistant** for current (hour) power forecast, total forecast for today and tomorrow, and a power-table array, with potential for use and display in graphs and automation.

![node-red-flow](/images/NodeRedFlowFSPV.png)

>[!NOTE]
> This uses my *personal* interpretation of the returned API data. I am not connected in any way with Forecast.Solar or the Home Assistant integration, and there are other interpretations for the details of the data returned from the API.

>[!CAUTION]
> This flow has been running on my Home Assistant (Node-RED addon) for two years, and every day I look at the daily graph to see how solar PV is performing against the forecast today, what the forecast looks like for tomorrow, and how it went yesterday. It very occasionally fails, but I find it a great tool for visually planning manual interventions for my home energy use. I don't use this information in automations as I find the accuracy can be out by more than +/- 20% too often. You are welcome to use this code as you wish, but it comes with absolutely no guarantees.

## What this Node-RED flow does

<details>
<summary> Flow action </summary>

This flow will trigger once every hour to:

- Read Forecast.Solar API for one or more solar planes, obtaining solar forecast figures for today and tomorrow
  - calculate timestamp dates for yesterday, today, and tomorrow
  - use API local and UTC time, and calculate timezone offset
  - maintain a context variable data object with the stored values
  - provide for HA sensor(s) for the current hour power forecast
  - record the actual solar production for each previous hour period

- Analyze the solar array
  - calculate PV forecast and timing details, for both today and tomorrow
  - build a 'power table' for discrete power levels, with start, end and duration
  - update full forecast every hour, retaining forecast history
  - provide an HA sensor with the forecast details for today and for tomorrow
  - provide an HA sensor with the full three day details for plotting to graph

</details>

## Prerequisites

> [!WARNING]
> If you are not comfortable running Node-RED then please do not attempt to use this code! This code uses JSONata throughout rather than function nodes and JavaScript. JSONata is a declarative-functional language. This may require more time and effort to understand, modify and debug the code *should you wish to make changes*.

<details>
<summary> Installation requirements </summary>

This is a **Node-RED** flow, written to run in Node-RED alongside Home Assistant. You will need:

- **Node-RED**, ideally running as a [Home Assistant add-on](https://github.com/hassio-addons/addon-node-red#readme)
- The API call *settings* for your own solar array (for the site and for each plane)
- If connecting back to Home Assistant, the [WebSocket nodes](https://github.com/zachowj/node-red-contrib-home-assistant-websocket) installed and their HA server configuration working correctly
- If recording actual solar production, a solar energy sensor in Home Assistant that tracks total energy produced (typically from your inverter)
- *No additional palette nodes are required*

</details>

## Forecast Solar

<details>
<summary> Forecast Solar and API calls </summary>

**Forecast Solar** is the base PV integration for Home Assistant, and ties in with the Energy Display to show a PV forecast for 'today', as well as providing several entities for forecast PV for various time periods. Multiple planes can be accommodated automatically, and the basic use is free of charge.

The data for Home Assistant is obtained by API call, processed and used immediately to populate the Energy graph and sensors, but is not retained or exposed in any way. Obtaining this data for other use (outside of using the provided sensor entities) requires using the API call directly.

**Forecast Solar** provide API calls to access PV forecasts. As a basic free-to-use plan, the simple forecast API is open and does not need any authorization. The calls do require precise location and orientation parameters.

You can find out more about [Forecast Solar API Documentation](https://doc.forecast.solar/).

The API call I am currently using is:

 `https://api.forecast.solar/estimate/{{{latitude}}}/{{{longitude}}}/{{{elevation}}}/{{{azimuth}}}/{{{power}}}`

The key elements in this are the site latitude and longitude, and the individual plane array elevation, azimuth and power. Additional further parameters can be added to include horizon shading and early/late damping.

Forecast Solar uses the PVGIS solar PV model, based on theoretical calculation and historic records for incident solar energy. This is modified at 15 minute intervals using weather projections from several sources. Combined with the precise solar array orientation, this provides an estimate of the array PV generation at future points in time.

> The flow maintains an array containing 72 hourly records for yesterday, today, and tomorrow. The forecast covers 48 records for today and tomorrow, and is updated every hour. The weather forecast will only change at most every 15 minutes. The 'free to use' account permits up to 12 API calls per hour, however there is little point in reading the API more than hourly.

### Hourly updates

**The forecast for today will change during the day.** Changes will take place *at most* every 15 minutes, but more likely only every few hours. The current-day forecast will change *even for times that have now past*. For this reason, at each hourly update, this flow takes a copy of the recent hour figure and saves to history, before updating the full two-day hourly array. The total day forecast, as the sum of the full day, can therefore change hourly, and it is a matter of choice whether to use the latest full-day forecast, or to use the history records *up to* 'now' together with the remaining forecast *from* 'now'. This flow, by default, *uses the historic records of the forecast where they exist* so as to provide a more stable day-forecast figure.

**The Forecast Solar site permits up to 12 API calls per hour on the 'free' account.** Each plane (for multiple planes) counts as one call. The hour appears to be 'rolling', but with an inherent overlap with regular hourly calls, a maximum of up to *four* planes can be accommodated. Additional calls (for testing etc.) can result in exceeding the call limit.

**API calls may fail**, however the data array will continue to hold the most recent forecast, and the flow will continue to collect and store the hourly actual production figures if required.

</details>

## Installing the flow

As is standard, the Node-RED flow is contained within a JSON file. The file contents can be copied, and imported using the usual Node-RED import from clipboard facility, or loaded directly from file.

> [!TIP]
> If you are updating from an earlier version, may I suggest that you disable all the flow entity sensor nodes and their corresponding sensor configuration nodes first! This 'shuts down' the old flow and reduces the risk of potential conflict! You may well see warning messages that you are importing nodes that already exist!

<details>
<summary> Installing the Node-RED code </summary>

> To avoid potential issues with an update, it may be worth taking a full backup of your existing flow, disabling the sensor and sensor-configuration nodes, redeploying and restarting Home Assistant and Node-RED, so as to remove the existing entity registrations in Home Assistant first. Then deleting the existing flow entirely before importing the update, so as to prevent duplication of the sensor or configuration nodes and the problems this can sometimes generate.

- Save the Node-RED flow (see the release file). In Node-RED go to the hamburger menu, select ‘import’, and import the JSON flow file.

- Set your site and plane parameters **before** attempting to use the API. You should ensure that location is within 100 metres, and that azimuth and elevation are accurate to the degree!

</details>

### Setting your parameters

**The API call parameters in the flow need to be hard-coded for your particular situation.**

<details>
<summary> Site and plane parameters </summary>

> Note that azimuth is measured as -180 for north, -90 for east, 0 for south and 90 for west. **This is different to the Home Assistant integration configuration settings!**

Full details on settings are provided in the [Forecast.Solar API documentation](https://doc.forecast.solar/api:estimate). The **site** change node holds the latitude and longitude parameters, and each individual **plane** change node holds a (unique) plane ID number, elevation and azimuth. Power is the individual plane array peak generating power in kW.

**For split arrays**, ensure that each individual array parameters are correct for those panels. If you have more than two arrays, add an additional timer and plane set of nodes and configure appropriately. For only one plane, remove or disable the additional nodes.

For fine adjustment, the site contains the additional **horizon** parameter. This is an example only, and should be either removed or corrected for your site. Note that the number of horizon elevation figures must be at least 12 and an integer divisor of 360. The morning and evening **dampening** parameters can be adjusted to correct further as required.

</details>

### Connection to Home Assistant

As good practice, the Home Assistant WebSocket nodes have been [scrubbed](https://zachowj.github.io/node-red-contrib-home-assistant-websocket/scrubber/) of their Home Assistant service configuration details, and *additionally disabled*. **You will need to edit these nodes, *and* the node configuration node behind each one, *and possibly the Home Assistant server configuration node itself.***

> [!IMPORTANT]  
> As a *minimum*, you will need to edit the sensor configuration nodes, at least to update and redeploy, in order to reconnect these configuration nodes with *your* default homeassistant server.

<details>
<summary> Connecting to Home Assistant </summary>

- If you are not already running a working connection with Home Assistant, make sure that you have installed the WebSocket nodes in the Node-RED palette, the [Node-RED companion integration](https://github.com/zachowj/hass-node-red) in Home Assistant, and you have correctly configured the *Home Assistant server configuration node*.
  - You should already be able to see a global context variable in Node-RED called `homeassistant` which contains a copy of the Home Assistant state.

In full detail:

- double click each Home Assistant Sensor node in turn to edit
  - enable the node (option bottom left)
  - click the edit pencil next to the *Entity config* entry to edit the ha-entity-config node
  - enable the ha-entity-config node (option bottom left)
  - check the *Server* entry - this should be HomeAssistant or similar
    - if you *do not* have a working HomeAssistant server, use the *Add new server...* option and set up the server
    - if you do have a working HomeAssistant server, ensure this is correctly selected in the *Server* entry field
  - update the *server* node, the *ha-entity-config* node, and finally save the *sensor* node and redeploy the flow once you have updated all the sensor nodes

![configuration-setup](/images/configsetup.png)

</details>

### Actual solar production

The flow permits the capture of the actual solar production, taken from an appropriate Home Assistant sensor. This latest version of the flow uses the **History** node to capture this value automatically, so no additional utility meter sensor is required.

> The value of the **actual solar energy production** recorded by the flow, is measured as the difference in total solar energy over a one hour period, from 1.5 hours ago to 0.5 hours ago. The reason for this is detailed further below.

<details>
<summary> Creating a solar energy sensor </summary>

Since solar PV inverters record the *power*, this must be converted to *energy* at some point for our use. Inverters will usually have a *total energy* (since first turned on) figure available, but typically only as whole kWh. Power as watts is also provided, and if this is measured (sampled) frequently enough, this can be used in a **Riemann Sum Integral** integration to sum instantaneous power over time to give energy.

**If you do not already have a solar energy sensor**, use the Home Assistant *Helper* to create a Riemann Sum, based on a solar power sensor. Ensure that the name and flow History node sensor name matches, use your *input sensor* of choice, set precision, metric prefix, and time period as shown. The integration method can be either Trapezoidal (for best results with an input sensor updated several time a minute) or Left (for sensors updated less often).

![Riemann Sum helper setup](/images/Riemann-integration-helper.png)

The sensor used to capture actual energy should show an increasing value of kWh to two decimal places, and in Home Assistant should look similar to:

![Solar total energy sensor](/images/solar-energy-sensor.png)

</details>

### Other optional settings

> [!CAUTION]
> The solar power array is passed to Home Assistant as a sensor attribute. The Home Assistant Recorder has a capacity limit and *may* generate an error message in the Home Assistant logs, to the effect that this entity state and attributes are too large and are not being saved by the Recorder.

There is no way to overcome this without reducing the size of the array. The error messages (if they appear) *are advisory* but can be prevented by adding the following to your Home Assistant configuration file. This optionally prevents the Recorder from attempting to save these particular sensors.

```yaml
recorder:
  exclude:
    entities:
      - sensor.fc_estimate_today
      - sensor.fc_estimate_tomorrow
      - sensor.fc_table
```

## Using this Node-RED flow

The flow should run without issue. Forecast Solar does have occasional maintenance outage periods (during Europe night). The flow here has no error checking or recovery - if the API calls fail then the call will be repeated at the next hour.

### When and how should I run the flow?

The flow is setup to run every hour. More frequently is unnecessary, and the flow could be updated only every two or three hours. The forecast does not change that often. Flow trigger here is using a standard inject node, to run hourly, and therefore it will use a cron expression and run at the top of the hour hh:00. To avoid running everything in my flows at the exact hour, and to avoid flooding the Forecast.solar API, I offset the run by using a 2 minute delay. Home Assistant runs the integration update hourly but randomly during each hour, and I would encourage you to be considerate to Forecast.solar in your use of this free service.

The flow maintains the context variables, adding a new 24 hour period and moving the data at the start of each new day. Where there is a longer period between 'today' and the stored dates, or where the array does not exist, a new blank array will be created. In cases of corrupt data simply delete the context variable and a new blank variable will be created at the next hourly API call.

To deal with Node-RED restarts, the main SolarFC (forecast) context variable should be stored using a file-based store. The SolarPT (power table) array is regenerated at each call and can be memory-based. All writes to and reads from context are presented in change nodes, allowing for easy selection of context store where multiple stores have been configured.

### Local time and Daylight Saving Time (DST)

<details>
<summary> How local time and DST is managed </summary>

The Forecast Solar API call uses the provided site location to identify the correct local time, which is returned as well as UTC time. By looking at the difference between local and UTC time, this flow calculates the timezone offset. Local time is used throughout.

Daylight Saving Time (DST) is treated by Forecast Solar as a switch. Once (at your location) DST is 'on', all the returned times are advanced by the DST change. This means that, for example, all times for today and tomorrow will be GMT, and then all times will switch to BST after the DST change has taken place. Since DST changes take place during the night, and by convention nighttime is when the sun does not shine, we do not have to be concerned with the change. *Note however that 'tomorrow' times will all be one hour incorrect during the day before DST time change*. If you wish to use the times for tomorrow, they will need to be adjusted for DST separately to this code. Good luck with that.

</details>

### What the figures mean IMPORTANT READING

There are several organizations providing solar forecast data. Interestingly, they universally shy away from explaining exactly what the data is that they return. Is it *power* or *energy* and what do the figures actually mean?

<details>
<summary> Power, energy, and history figures explained </summary>

We talk about solar power and solar energy almost as if they were the same thing. They are not. Energy is the ability to do work, and is measured in Joules. Power is the rate at which work is being done, and is therefore Joules per second, or more conveniently Watts. When we are dealing with lots of power and energy, we typically use kilo-Joules and kilo-Watts (kW) to be more convenient, and when dealing with time we use hours rather than seconds.

Electricity, for practical reasons, is measured as power in kW, however electrical energy is back-measured as Watt Hours (Wh or kWh) rather than Joules. Starting with instantaneous power (W) we need to sum the value at each second, over a full hour, to arrive at the Watt-hour energy value.

If we look at a power-time graph, then the energy used or generated is the area under the graph, and this is what the Riemann Sum is approximating, by summing up lots of little rectangles of height as power (W) multiplied by width as time period (seconds) and factoring by 1000 and 3600 to get kWh.

So, back to the solar forecast.
The Forecast.solar API returns, for each hour, a power and a Watt-hour (energy) measurement. There has been quite a bit of debate on this (which you can read up on in the Home Assistant community forum) as to what the power figure actually means. Here is my *personal* take on this (you make up your own mind please).

The power figure given is the forecasted instantaneous power in Watts at the precise hour. As a continuous variable, power can be plotted on a time graph as a line.

The energy figures have to be calculated over a period of time, which is usually a one-hour period. From a simple power graph, we can estimate the energy by taking the average power (between two consecutive hourly figures). Since we are working over a one hour period, the average power value over the hour is numerically equivalent to the energy Watt-hour (Wh) value.

![Power or energy?](/images/Power-or-energy.png)

In this example are shown the rounded forecast figures for one of my planes (as I write this today). The forecast power figures are for the hour, so at 09:00 I expect 1100 Watts. From the graph line, I can interpolate, and I therefore expect 1150 Watts at 09:30 (half way between 1100 and 1200).

The energy is the area under the line, and so the total forecast energy between 10:00 and 11:00 is equal to the average power over the hour. This is 1325 Wh (1.33 kWh).

The problem comes when I try to plot both power and energy on the same graph. Since energy is over a period, it should be shown as a bar chart, spanning the hour and with height representing the energy. This is the orange bar shown. Problem is, this should be plotted centred on the mid point. Graphs will plot either the bar or a line at a point, so the plot should be at 10:30. As I want to make life simple, I want to plot both power and energy together, so I either have to move the power to a calculated half-hour midpoint, or measure the actual energy centred on the hour, or plot a dual-graph with one series at the hour and another at the half hour.

I chose to measure and plot energy centred on the hour. This means, looking at the green bar, for the plot at 09:00, the measured actual energy is from 08:30 to 09:30. The Node-RED flow is going to be triggered sometime after 10:00, when the most recent hour is 10:00, and the actual energy is then measured from 08:30 to 09:30 and plotted at 09:00. It works.

The remaining issue is that, as plotted as a simple line graph, forecast power and actual energy don't quite equate. You can see that, for the orange bar the centre plot point equals the power line at that point. However, the green bar centre point does not. To deal with this, the flow also calculates an energy figure for the period hour-30 minutes to hour+30 minutes, so that forecast energy equivalent can be plotted at the hour. For anyone still reading this, this is the average power between 08:30 and 09:30, and the average power at 08:30 is the average power between 08:00 and 09:00, thus the average of the average powers for 09:00 is [P(08:00) + 2 * P(09:00) + P(10:00)] / 4. Oh yes, it is. In practice, the forecast power plot is almost identical to the calculated energy forecast plot.

### History of the power forecast

To make life interesting, the forecast is updated for the entire day and continues to be updated even as the day progresses. This can mean that the 'forecast' becomes a more accurate 'back-cast', but it also can mean that the day forecast can become vastly incorrect. Since the total day power figure is just the sum of the total of the day, the day forecast can change. To prevent the historical part of the forecast from impacting daily figures, the flow captures each forecast value at the last hour, and copies this to a history record. This permits the flow to use history values rather than back-cast values so as to provide more stability.

The flow therefore keeps, by hour

- the forecast power at the hour given
- a historical capture of the forecast power, taken at the hour earlier
- the calculated forecast hourly energy, across the period around the hour given
- the actual measured hourly energy, across the period around the hour given

</details>

### Context variables

<details>
<summary> Details of all the flow context variables </summary>

The context store read and writes are all performed in Change nodes. This facilitates more easily selecting a different store than 'default' if you have multiple context stores and wish to make any of the context variables persistent.

#### SolarFC

The Solar Forecast object which holds an array with 72 hourly records for yesterday, today, and tomorrow with fields for:

- the main array, with forecast, forecast history, forecast energy, and actual energy (Watts) for each hour
- the dates used, timezone offset, and last update timestamp
- today and tomorrow solar day times, total forecast energy, and the incremental day difference for tomorrow
- the saved most recent API forecast power results, by plane

This is created on first API call, day-shifted each new day, and updated on every API call. The dates are also recalculated on change of timezone offset at DST.

**Note** that the forecast energy in this variable is the day Watt-hour value provided by the API call, and reflects the current forecast.

#### SolarPT

Holds an object for today, and for tomorrow, each with fields for:

- date (today / tomorrow)
- solar day start, end, and duration in minutes
- the hour that is the start of the first and last full hour of the day
- total forecast energy for the day
- maximum power (from hour figures)
- an array of one or more maximum (hour) points
- an array of zero or more minimum (hour) points
- a power table array
- power table analysis parameters used

This object is updated on successful API update.

**Note** that the forecast energy in this variable is calculated from the history-forecast maintained in the solar table. This will be slightly different to the API total day energy forecast, and will typically change by less during the day.

**The power table** is regenerated at each API update, and contains time period details for given intervals of forecast power. The flow uses parameters to create an array at, for example, set 100 Watt intervals (from 0 to 4000). Then, the forecast is analyzed and for each power level, the corresponding period of time at that power is calculated by interpolation.

Each power level holds the power level and a count of the corresponding time periods. Mostly, this count will be 1, but for power plots with minima, upper power levels may have two or even more periods.

Each time period (in each level) contains a start time and stop time, as "hh:mm" strings, and a duration in minutes.

Note that the power level zero array[0] contains the solar day start and end time and period as given by the Forecast.solar API call return. Each array item is either empty (count = 0) or set (count > 0) and will contain that time period(s) for which the forecast power is equal to or greater than the power level of that entry.

The analysis uses the historical forecast values by default, and reports this "oldvalue" and the used power level interval (100) in the context variable.

</details>

### Home Assistant Sensors

#### Plane Power Forecast

<details>
<summary> Forecast Power</summary>

The *state value* is the forecasted power at the current hour.

Attributes hold the forecasted power at the next hour, and also an array of interpolated power levels by minute. The forecast power at, for example, hh:20 would be the entity `state + attributes.minutes[20]`. Saves you having to work it out.

</details>

#### FC Estimate Today/Tomorrow

<details>
<summary> Forecast Estimate </summary>

The *state value* is the forecasted total energy for today/tomorrow in kWh.

Attribute fields hold:

- the date
- the power table array
- solar day start and stop times
- arrays for maximum and minimum hours
- max power (kW) for the day

</details>

#### FC Solar Table

<details>
<summary> Forecast solar table (for graph) </summary>

The *state value* is a date and time of the last hour, and is always local time. The attributes hold constructed arrays for the time (hours) and the forecast, history, energy, and actual energy figures. This is constructed for direct use in Home Assistant Apex-Charts graph. Note that array values for history, energy, and actual values will be `null` for the current hour and future records.

</details>

## Using the power table

The power table is an array held in the SolarPT context variable, and passed to Home Assistant in the Estimated Forecast entities, one entity for 'today' and one for 'tomorrow'.

<details>
<summary> Suggestions for using the solar forecast power table </summary>

Both tables (today/tomorrow) are recalculated at each API call, and therefore a user may wish to take a copy of the table for today at the start of the solar day, for use in automations during the day. In the UK (Europe) my assessment is that the day-head forecast is both reasonably accurate and stable at around 04:00 or 05:00. Since the forecast can change during the day, the power table will also change.

In making decisions based on the power forecast, there are three parameters:

- The power required (eg more than 900 W)
- The time required (eg after 08:00 and before 15:00)
- The duration required (eg for at least 90 minutes)

In can become both complex and self-defeating to ask too many restrictions. Further complications arise when the forecast also contains one or more minima during the day, since higher power levels then have discontinuous periods.

By way of example, the following small flow will read in the power table for today from context and select an array of power/times that meet the parameters. In the example shown, for power greater than 600 Watts, starting after 08:00 and for at least 60 minutes, three results are returned, and it is a simple matter to select just the first in the array.

![Flow to read and use power table](/images/use_power_table.png)

This uses just the inject node with some JSONata. Although perhaps challenging to learn and use, JSONata is very powerful and it would take a great deal of template YAML to replicate in Home Assistant. [I have tried, and this is not a challenge, but you are *most* welcome to post your YAML equivalent.]

The JSONata required here is:

```JSONata
(
    $pa:=powertable[count>0].{
        "P": power,
        "C": count,
        "S": (period.start)[0],
        "E": (period.stop)[-1],
        "M": $sum(period.minutes)};

    $pa[P>=$$.parms.Power and S>=$$.parms.Start and M>=$$.parms.Length].
        {"From": S, "Power": P & " for " & M & " minutes"}
  
)
```

</details>

## Display

### Graphs

The flow includes a Node-RED dashboard graph.

![Node-RED dashboard graph](/images/Node-RED-forecast-graph.png)

For display in Home Assistant, I use the custom [apexcharts-card](https://github.com/RomRider/apexcharts-card). The graph configuration used here shows the forecast figures over a complete 72 hour period for 'yesterday', 'today' and 'tomorrow'.

![Apex Charts Graph](/images/HA_Apex_Charts_graph.png)

<details>
<summary> Forecast graph configuration </summary>

Card settings:

Graph_span is set to 3 days, with span to end at the end of the day offset by +1day so the graph runs from yesterday to tomorrow in full.

Data series are all a smooth line, using a data generator. This takes the *attribute.array* and pulls the *fchours* (for the timestamp) and *fcwatts* (forecast power), *fcold* (forecast power history), *fcactual* (actual energy), and *fcwh* (forecast energy) mapping from the full array to the new time/price array required for the chart.

Configuration code is given below:

```yaml
type: custom:apexcharts-card
graph_span: 3d
span:
  end: day
  offset: +1d
header:
  show: true
  title: 'Solar Forecast: yesterday - today - tomorrow'
now:
  show: true
  label: now
show:
  last_updated: true
series:
  - entity: sensor.fc_table
    data_generator: |
      return entity.attributes.fchours.map((fchr, index) => {
        return [new Date(fchr).getTime(), entity.attributes.fcwatts[index]];
      });
    curve: smooth
    name: Forecast
    show:
      in_header: false
      legend_value: false
    stroke_width: 2
    color: orange
  - entity: sensor.fc_table
    data_generator: |
      return entity.attributes.fchours.map((fchr, index) => {
        return [new Date(fchr).getTime(), entity.attributes.fcold[index]];
      });
    curve: smooth
    name: History
    show:
      in_header: false
      legend_value: false
    stroke_width: 2
    color: magenta
  - entity: sensor.fc_table
    data_generator: |
      return entity.attributes.fchours.map((fchr, index) => {
        return [new Date(fchr).getTime(), entity.attributes.fcactual[index]];
      });
    curve: smooth
    name: Actual
    show:
      in_header: false
      legend_value: false
    stroke_width: 2
    color: blue
  - entity: sensor.fc_table
    data_generator: |
      return entity.attributes.fchours.map((fchr, index) => {
        return [new Date(fchr).getTime(), entity.attributes.fcwh[index]];
      });
    curve: smooth
    name: Energy
    show:
      in_header: false
      legend_value: false
    stroke_width: 2
    color: green
```

Note that for Home Assistant dark display modes, other colours might be more appropriate.

Alternative graph options can be used. Here is a graph for just today, with the forecast and history power values plotted as straight lines, and the forecast and actual energy plotted as a bar. The graph span has been curtailed to just 17 hour (enough for maximum solar day for central European latitudes where I live) with a day end offset of -2 hours to make the graph plot from 05:00 to 22:00.

![day solar forecast graph plot](/images/day-solar-forecast.png)

**Notes**:

- In Apex Charts, series plots can be 'turned off' by clicking on the legend, and here the 'energy' series has been suppressed to better show how the actual energy (plotted as a bar) aligns with the hour and the power series plotting.
- The forecast history shows how the current forecast for 10:00 has changed.
- The 'now' indicator is just past the current hour of 14:00, with the most recently updated history and actual for 13:00. The actual values for 14:00 range from 13:30 to 14:30, and are therefore yet to be calculated at this point in time.
- The forecast sensor values for the estimate for today, and for tomorrow, are shown in the entity card below the graph, together with the last update timestamp.
- The forecast today/tomorrow values (total energy) additionally show the last change time, and you will see that whilst the forecast for tomorrow changed at the 14:00 API call, the forecast for today last changed at the 11:00 update. This is reinforced by the fact that the forecast line and the forecast-history line do not match up to 10:00, but do from 11:00.

Configuration for this one-day graph is given below:

```yaml
type: custom:apexcharts-card
graph_span: 17h
span:
  end: day
  offset: '-2h'
header:
  show: true
  title: 'Solar Forecast for Today'
now:
  show: true
  label: now
show:
  last_updated: true
series:
  - entity: sensor.fc_table
    data_generator: |
      return entity.attributes.fchours.map((fchr, index) => {
        return [new Date(fchr).getTime(), entity.attributes.fcwatts[index]];
      });
    curve: straight
    name: Forecast
    show:
      in_header: false
      legend_value: false
    stroke_width: 3
    color: orange
  - entity: sensor.fc_table
    data_generator: |
      return entity.attributes.fchours.map((fchr, index) => {
        return [new Date(fchr).getTime(), entity.attributes.fcold[index]];
      });
    curve: straight
    name: History
    show:
      in_header: false
      legend_value: false
    stroke_width: 3
    color: magenta
  - entity: sensor.fc_table
    data_generator: |
      return entity.attributes.fchours.map((fchr, index) => {
        return [new Date(fchr).getTime(), entity.attributes.fcactual[index]];
      });
    type: column
    curve: smooth
    name: Actual
    show:
      in_header: false
      legend_value: false
    stroke_width: 1
    color: blue
  - entity: sensor.fc_table
    data_generator: |
      return entity.attributes.fchours.map((fchr, index) => {
        return [new Date(fchr).getTime(), entity.attributes.fcwh[index]];
      });
    type: column
    curve: straight
    name: Energy
    show:
      in_header: false
      legend_value: false
    stroke_width: 1
    color: green
```

</details>

## Trouble shooting

If you are having problems...

<details>
<summary> Basic trouble shooting guide... </summary>

### It does not work

**The API call?** You can always make up a URL with your settings, post this to a browser address bar, and the API call will return either a JSON result or an error message. Remember - you only have 12 free calls per hour.

**The entities in Home Assistant?** Check your Node-RED Home Assistant server configuration.

### Forecast power is widely incorrect

Check your parameters. Accuracy is essential, so don't guess.

### My actual does not line up with the forecast

This is a subject of *much* debate.

Examine the returned data - the start and end timestamps are the start and end of the solar day, and should agree with your local experience. Check your settings, and if necessary add a horizon and damping. Check your timezone matches the returned local time from the API call.

### The forecast is not accurate

It is a *forecast*, and you should only expect an accurate forecast and a good match to your actual solar generation on cloudless sunny days. The clear-sky prediction (with the correct parameters) on sunny days will be sound.

After that, weather happens. Solar generation is roughly 15% of system peak from diffuse light, and 85% from direct incident sunlight. Particles and moisture in the atmosphere reduce, and dark cloud can block both direct and diffuse sunlight. Shading on any part of an array can have a major impact without micro-inverters.

Note that, the further away from south a solar plane faces, the more unreliable the forecast becomes. Planes that face northerly from direct east or west appear to be particularly unreliable at the start or end of the solar day.

</details>

## JSONata

Is a very different language, so if you are looking to either understand the code used here, or to use and modify, then you are probably going to have to learn a bit about JSONata.

The JSONata documentation can be found [here](https://docs.jsonata.org/overview.html)

There is a great [sandbox](https://try.jsonata.org/) which I use for all my development work.

## Buy me a coff*ee*?

I hope that you like this. I wrote it just for fun (seriously). I am retired, and they said 'learn a language, it will keep your brain active'. I ended up learning JSONata. Brain overload.

I am no longer working, so my time is my own. If you have a problem, want to ask a question, need some help, then please ask, but *please* don't expect me to jump just for you!

People seem to go for 'buy me a coff*ee*'. I don't need more coffee - and this is just a hobby. I guess I could start 'buy me a coff*in*' as it would be of more practical use (joke-*noir*) but hey let's look on the bright side of life while we still can! Buy *yourself* a coffee. Buy *someone you know* a coffee. Buy a *random stranger* a coffee. Buy [Knut](https://forecast.solar/about.html) a coffee for all the work he does on the Forecast Solar (without which you would *not* be reading this). Put the money in a jar and save it - one coffee a day for forty years adds up to a lot, I know, I saved that money so that I might have a comfortable retirement and be able to buy *me* a coffee *myself*!

Good luck and best wishes with your Solar PV project!
