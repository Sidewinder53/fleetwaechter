<script lang="ts">
  import 'carbon-components-svelte/css/g10.css';
  import {
    Accordion,
    AccordionItem,
    CodeSnippet,
    Column,
    Content,
    DatePicker,
    DatePickerInput,
    Dropdown,
    Grid,
    Header,
    LocalStorage,
    NumberInput,
    Row,
    Tile
  } from 'carbon-components-svelte';
  import '@carbon/styles/css/styles.css';
  import '@carbon/charts/styles.css';
  import { GaugeChart, LineChart } from '@carbon/charts-svelte';
  import type { LineChartOptions, GaugeChartOptions, ScaleTypes } from '@carbon/charts/interfaces';

  import {
    differenceInMonths,
    differenceInCalendarDays,
    startOfTomorrow,
    getDate,
    getDaysInMonth,
    lastDayOfMonth,
    parse,
    setDate,
    subMonths
  } from 'date-fns';

  let carClasses = [
    {
      name: 'upTo250',
      title: 'Up to 250 metric horse power 🚙',
      surchargeLower: 0.19,
      surchargeHigher: 0.29,
      limit: 4000
    },
    {
      name: 'moreThan250',
      title: 'More than 250 metric horse power 🏎️',
      surchargeLower: 0.29,
      surchargeHigher: 0.39,
      limit: 4000
    }
  ];

  let currentKm = 0;
  let initialKm = 0;
  let allowanceMonthly = 0;
  let checkOutValue = '';
  let checkInValue = '';
  let carClass = 0;
  let currentDay = startOfTomorrow();
  $: configurationComplete = (
    (currentKm !== 0) &&
    (initialKm !== 0) &&
    (allowanceMonthly !== 0) &&
    (checkOutValue !== '') &&
    (checkInValue !== '')
  );

  $: checkOutDate = parse(checkOutValue, 'dd/MM/yyyy', new Date());
  $: checkInDate = parse(checkInValue, 'dd/MM/yyyy', new Date());
  $: checkedOutDaysTotal = differenceInCalendarDays(checkInDate, checkOutDate);
  $: checkedOutDaysUntilToday = differenceInCalendarDays(currentDay, checkOutDate);

  // Calculations
  $: allowanceFactorMonthCO = 1 - (getDate(checkOutDate) - 1) / getDaysInMonth(checkOutDate);
  $: allowanceFactorMonthCI = getDate(checkInDate) / getDaysInMonth(checkInDate);
  $: allowanceMonthCO = allowanceMonthly * allowanceFactorMonthCO;
  $: allowanceMonthCI = allowanceMonthly * allowanceFactorMonthCI;

  // Calculate number of full months by skipping the CO and CI months entirely
  $: fullMonths = differenceInMonths(
    setDate(subMonths(checkInDate, 1), 1),
    lastDayOfMonth(subMonths(checkOutDate, 1))
  );

  $: allowanceTotal = allowanceMonthly * fullMonths + allowanceMonthCO + allowanceMonthCI;
  $: allowanceUsed = currentKm - initialKm;
  $: usedAllowancePercentage = ((currentKm - initialKm) / allowanceTotal) * 100;
  $: allowanceRemaining = allowanceTotal - allowanceUsed;

  $: allowanceAvailablePerDay = allowanceTotal / checkedOutDaysTotal;
  $: allowanceUsedPerDay = allowanceUsed / checkedOutDaysUntilToday;
  $: withinDailyAllowance = allowanceUsedPerDay <= allowanceAvailablePerDay;
  $: dailyAllowanceOveragePercentage = (allowanceUsedPerDay / allowanceAvailablePerDay - 1) * 100;

  $: totalAllowanceGaugeColor = determineColor(usedAllowancePercentage);

  let totalAllowanceGaugeOptions: GaugeChartOptions;
  $: totalAllowanceGaugeOptions = {
    title: 'Total Allowance',
    resizable: false,
    height: '150px',
    width: '100%',
    toolbar: {
      enabled: false
    },
    color: {
      scale: {
        value: totalAllowanceGaugeColor
      }
    }
  };

  $: totalAllowanceGaugeData = [
    {
      group: 'value',
      value: Math.round(usedAllowancePercentage)
    }
  ];

  $: dailyAllowanceGaugeColor = determineColor(100 + dailyAllowanceOveragePercentage);

  let dailyAllowanceGaugeOptions: GaugeChartOptions;
  $: dailyAllowanceGaugeOptions = {
    title: 'Daily Allowance',
    resizable: false,
    height: '150px',
    width: '100%',
    toolbar: {
      enabled: false
    },
    color: {
      scale: {
        value: dailyAllowanceGaugeColor
      }
    }
  };

  $: dailyAllowanceGaugeData = [
    {
      group: 'value',
      value: Math.round(100 + dailyAllowanceOveragePercentage)
    }
  ];

  let extrapolatedAllowanceChartOptions: LineChartOptions = {
    title: 'Allowance Extrapolation',
    height: '250px',
    axes: {
      bottom: {
        mapsTo: 'key',
        scaleType: 'time' as ScaleTypes.TIME
      },
      left: {
        mapsTo: 'value',
        title: 'Kilometers',
        scaleType: 'linear' as ScaleTypes.LINEAR,
        includeZero: false
      }
    },
    toolbar: {
      enabled: false
    }
  };

  $: extrapolatedAllowanceChartData = [
    {
      group: 'Planned',
      key: checkOutDate,
      value: initialKm
    },
    {
      group: 'Planned',
      key: checkInDate,
      value: initialKm + allowanceTotal
    },
    {
      group: 'Extrapolated',
      key: checkOutDate,
      value: initialKm
    },
    {
      group: 'Extrapolated',
      key: checkInDate,
      value: initialKm + allowanceTotal * (1 + dailyAllowanceOveragePercentage / 100)
    },
    {
      group: 'Current',
      key: currentDay,
      value: currentKm
    }
  ];

  $: debugString = `CO Date: ${checkOutDate.toDateString()} - applied factor: ${allowanceFactorMonthCO.toFixed(2)} (amounts to ${allowanceMonthCO.toFixed(2)} km)
CI Date: ${checkInDate.toDateString()} - applied factor: ${allowanceFactorMonthCI.toFixed(2)} (amounts to ${allowanceMonthCI.toFixed(2)} km)
Months with full allowance: ${fullMonths} (amounts to ${allowanceMonthly * fullMonths} km)
Total allowance: ${allowanceTotal.toFixed(2)} km
Checked out days until today: ${checkedOutDaysUntilToday}
Checked out days in total: ${checkedOutDaysTotal}
Within daily allowance: ${withinDailyAllowance}
Factor daily allowance used: ${dailyAllowanceOveragePercentage.toFixed(2)}
Assumed current day (start of tomorrow): ${currentDay.toDateString()}`;

  function determineColor(percentage: number) {
    if (percentage <= 85) {
      return 'green';
    } else if (percentage <= 95) {
      return 'orange';
    } else {
      return 'red';
    }
  }
</script>

<svelte:head>
  <title>Fleetwächter</title>
</svelte:head>

<LocalStorage key="currentKm" bind:value={currentKm} />
<LocalStorage key="initialKm" bind:value={initialKm} />
<LocalStorage key="monthlyKmAllowance" bind:value={allowanceMonthly} />
<LocalStorage key="checkOutDate" bind:value={checkOutValue} />
<LocalStorage key="checkInDate" bind:value={checkInValue} />
<LocalStorage key="carClass" bind:value={carClass} />

<Header platformName="Fleetwächter" expandedByDefault={false} expansionBreakpoint={320} />

<Content>
  <Grid>
    <Row padding>
      <Column>
        <NumberInput size="xl" hideSteppers label="Current kilometers" helperText="The amount of kilometers on the cars odometer" bind:value={currentKm} />
      </Column>
    </Row>
    <Row>
      <Column>
        <Accordion>
          <AccordionItem title="Result" open={configurationComplete} disabled={!configurationComplete} >
            <Row>
              <Column sm={4} md={4} lg={5}>
                <Tile>
                  <GaugeChart
                    bind:data={totalAllowanceGaugeData}
                    bind:options={totalAllowanceGaugeOptions} />
                  <h5>Used allowance:</h5>
                  <p>{allowanceUsed.toFixed(1)} km</p>
                  <h5>Remaing allowance:</h5>
                  <p>{allowanceRemaining.toFixed(1)} km</p>
                </Tile>
              </Column>
              <Column sm={4} md={4} lg={5}>
                <Tile>
                  <GaugeChart
                    bind:data={dailyAllowanceGaugeData}
                    bind:options={dailyAllowanceGaugeOptions} />
                  <h5>Average daily driven:</h5>
                  <p>{allowanceUsedPerDay.toFixed(1)} km</p>
                </Tile>
              </Column>
              <Column sm={4} md={4} lg={5}>
                <Tile>
                  <LineChart
                    bind:data={extrapolatedAllowanceChartData}
                    options={extrapolatedAllowanceChartOptions} />
                </Tile>
              </Column>
            </Row>
          </AccordionItem>
          <AccordionItem title="Configure kilometer allowance" open={!configurationComplete}>
            <Grid>
              <Row>
                <Column padding sm={4} md={8} lg={8}>
                  <DatePicker
                    datePickerType="single"
                    dateFormat="d/m/Y"
                    on:change
                    bind:value={checkOutValue}>
                    <DatePickerInput labelText="Check out date (when you got the car)" />
                  </DatePicker>
                </Column>
                <Column padding sm={4} md={8} lg={8}>
                  <DatePicker
                    datePickerType="single"
                    dateFormat="d/m/Y"
                    on:change
                    bind:value={checkInValue}>
                    <DatePickerInput labelText="Check in date (when you return the car)" />
                  </DatePicker>
                </Column>
              </Row>
              <Row>
                <Column padding sm={4} md={4} lg={8}>
                  <NumberInput
                    hideSteppers
                    label="Monthly kilometer allowance"
                    bind:value={allowanceMonthly} />
                </Column>
                <Column padding sm={4} md={4} lg={8}>
                  <NumberInput
                    hideSteppers
                    label="Initial kilometers (at check out)"
                    bind:value={initialKm} />
                </Column>
                <Column padding sm={4} md={8} lg={16}>
                  <Dropdown
                    titleText="Car class"
                    bind:selectedId={carClass}
                    items={carClasses.map((carClass, index) => ({
                      id: index,
                      text: carClass.title
                    }))}
                    let:item
                    let:index>
                    <div>
                      <strong>{carClasses[index].title}</strong>
                    </div>
                    <div>
                      Lower: {carClasses[index].surchargeLower}€ - Higher: {carClasses[index]
                        .surchargeHigher}€ - Limit: {carClasses[index].limit} km
                    </div>
                  </Dropdown>
                </Column>
              </Row>
            </Grid>
          </AccordionItem>
          <AccordionItem title="Debug information" open={false}>
            <CodeSnippet wrapText type="multi" hideCopyButton={true} code={debugString} expanded={true} />
          </AccordionItem>
        </Accordion>
      </Column>
    </Row>
  </Grid>
</Content>

<style>
  :global(.bx--list-box__menu-item, .bx--list-box__menu-item__option) {
    height: auto;
  }

  :global(.bx--content) {
    padding: 0.5rem;
  }

  :global(.bx--col) {
    padding-right: 0.25rem;
    padding-left: 0.75rem;
  }
</style>
