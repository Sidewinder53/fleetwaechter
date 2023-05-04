<script lang="ts">
  import 'carbon-components-svelte/css/g10.css';
  import {
    Accordion,
    AccordionItem,
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
  import type { LineChartOptions, ScaleTypes } from '@carbon/charts/interfaces';

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

  $: totalAllowanceGaugeData = [
    {
      group: 'value',
      value: Math.round(usedAllowancePercentage)
    }
  ];

    let extrapolatedAllowanceChartOptions: LineChartOptions = {
              title: 'Allowance Extrapolation',
              height: '250px',
              axes: {
                bottom: {
                  mapsTo: 'key',
                  scaleType: "time" as ScaleTypes.TIME
                },
                left: {
                  mapsTo: 'value',
                  title: 'Kilometers',
                  scaleType: "linear" as ScaleTypes.LINEAR,
                  includeZero: false,
                }
              },
              toolbar: {
                enabled: false
              }
            }

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
</script>

<LocalStorage key="currentKm" bind:value={currentKm} />
<LocalStorage key="initialKm" bind:value={initialKm} />
<LocalStorage key="monthlyKmAllowance" bind:value={allowanceMonthly} />
<LocalStorage key="checkOutDate" bind:value={checkOutValue} />
<LocalStorage key="checkInDate" bind:value={checkInValue} />
<LocalStorage key="carClass" bind:value={carClass} />

<Header platformName="Fleetwächter" expandedByDefault={false} expansionBreakpoint={320} />

<Content>
  <Grid>
    <Row>
      <Column>
        <Tile>
          <GaugeChart
            bind:data={totalAllowanceGaugeData}
            options={{
              title: 'Used allowance (total)',
              resizable: false,
              height: '250px',
              width: '100%',
              toolbar: {
                enabled: false
              }
            }} />
        </Tile>
      </Column>
      <Column>
        <Tile>
          <h5>Used allowance:</h5>
          <p>{allowanceUsed} km</p>
          <h5>Remaing allowance:</h5>
          <p>{allowanceRemaining} km</p>
          {#if !withinDailyAllowance}
            <h5>Overstepped daily allowance by:</h5>
            <p>{dailyAllowanceOveragePercentage.toFixed(1)} %</p>
          {/if}
        </Tile>
      </Column>
      <Column>
        <Tile>
          <LineChart bind:data={extrapolatedAllowanceChartData} options={extrapolatedAllowanceChartOptions} />
        </Tile>
      </Column>
    </Row>
  </Grid>
  <Grid>
    <Row>
      <Column>
        <NumberInput hideSteppers label="Current kilometers" bind:value={currentKm} />
        <Accordion>
          <AccordionItem title="Configure kilometer allowance" open={true}>
            <Grid>
              <Row>
                <Column>
                  <DatePicker
                    datePickerType="single"
                    dateFormat="d/m/Y"
                    on:change
                    bind:value={checkOutValue}>
                    <DatePickerInput labelText="Check out date (when you got the car)" />
                  </DatePicker>
                </Column>
                <Column>
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
                <Column>
                  <NumberInput
                    hideSteppers
                    label="Monthly kilometer allowance"
                    bind:value={allowanceMonthly} />
                </Column>
                <Column>
                  <NumberInput
                    hideSteppers
                    label="Initial kilometers (at check out)"
                    bind:value={initialKm} />
                </Column>
              </Row>
              <Row>
                <Column>
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
        </Accordion>
      </Column>
    </Row>
    <br />
    <Row>
      <h3>Debug information</h3>
    </Row>
    <Row>
      This information may be used to debug calculations.
      <br /><br />
    </Row>
    <Row>
      <p>
        CO Date: {checkOutDate} - applied factor: {allowanceFactorMonthCO} (amounts to {allowanceMonthCO}
        km)<br />
        CI Date: {checkInDate} - applied factor: {allowanceFactorMonthCI} (amounts to {allowanceMonthCI}
        km)<br />
        Months with full allowance: {fullMonths} (amounts to {allowanceMonthly * fullMonths} km)<br />
        Total allowance: {allowanceTotal} km<br />
        Checked out days until today: {checkedOutDaysUntilToday}<br />
        Checked out days in total: {checkedOutDaysTotal}<br />
        Within daily allowance: {withinDailyAllowance}<br />
        Factor daily allowance used: {dailyAllowanceOveragePercentage}<br />
      </p>
    </Row>
  </Grid>
</Content>

<style>
  :global(.bx--list-box__menu-item, .bx--list-box__menu-item__option) {
    height: auto;
  }
</style>
