<script>
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
    HeaderNav,
    HeaderNavItem,
    HeaderNavMenu,
    LocalStorage,
    NumberInput,
    Row,
    SkipToContent
  } from 'carbon-components-svelte';

  import {
    differenceInMonths,
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

  $: checkOutDate = parse(checkOutValue, 'dd/MM/yyyy', new Date());
  $: checkInDate = parse(checkInValue, 'dd/MM/yyyy', new Date());

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
</script>

{@debug allowanceFactorMonthCI}

<LocalStorage key="currentKm" bind:value={currentKm} />
<LocalStorage key="initialKm" bind:value={initialKm} />
<LocalStorage key="monthlyKmAllowance" bind:value={allowanceMonthly} />
<LocalStorage key="checkOutDate" bind:value={checkOutValue} />
<LocalStorage key="checkInDate" bind:value={checkInValue} />
<LocalStorage key="carClass" bind:value={carClass} />

<Header platformName="Fleetwächter" expandedByDefault={false} expansionBreakpoint={320}>
  <!-- <svelte:fragment slot="skip-to-content">
    <SkipToContent />
  </svelte:fragment>
  <HeaderNav>
    <HeaderNavItem href="/" text="Link 1" />
    <HeaderNavItem href="/" text="Link 2" />
    <HeaderNavItem href="/" text="Link 3" />
    <HeaderNavMenu text="Menu">
      <HeaderNavItem href="/" text="Link 1" />
      <HeaderNavItem href="/" text="Link 2" />
      <HeaderNavItem href="/" text="Link 3" />
    </HeaderNavMenu>
  </HeaderNav> -->
</Header>

<Content>
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
                    bind:value={checkOutValue}
                  >
                    <DatePickerInput labelText="Check out date (when you got the car)" />
                  </DatePicker>
                </Column>
                <Column>
                  <DatePicker
                    datePickerType="single"
                    dateFormat="d/m/Y"
                    on:change
                    bind:value={checkInValue}
                  >
                    <DatePickerInput labelText="Check in date (when you return the car)" />
                  </DatePicker>
                </Column>
              </Row>
              <Row>
                <Column>
                  <NumberInput
                    hideSteppers
                    label="Monthly kilometer allowance"
                    bind:value={allowanceMonthly}
                  />
                </Column>
                <Column>
                  <NumberInput
                    hideSteppers
                    label="Initial kilometers (at check out)"
                    bind:value={initialKm}
                  />
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
                    let:index
                  >
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
        Months with full allowance: {fullMonths} (amounts to {allowanceMonthly * fullMonths} km)<br
        />
        Total allowance: {allowanceTotal} km
      </p>
    </Row>
  </Grid>
</Content>

<style>
  :global(.bx--list-box__menu-item, .bx--list-box__menu-item__option) {
    height: auto;
  }
</style>
