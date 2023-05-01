<script>
  import 'carbon-components-svelte/css/g10.css';
  import {
    Header,
    Content,
    Grid,
    Row,
    Column,
    NumberInput,
    DatePicker,
    DatePickerInput,
    Accordion,
    AccordionItem,
    LocalStorage,
    SkipToContent,
    HeaderNav,
    HeaderNavItem,
    HeaderNavMenu
  } from 'carbon-components-svelte';

  let currentKm = 0;
  let initialKm = 0;
  let monthlyKmAllowance = 0;
  /**
   * @type {Date}
   */
  let checkOutDate;
  /**
   * @type {Date}
   */
  let checkInDate;
</script>

<LocalStorage key="currentKm" bind:value={currentKm} />
<LocalStorage key="initialKm" bind:value={initialKm} />
<LocalStorage key="monthlyKmAllowance" bind:value={monthlyKmAllowance} />
<LocalStorage key="checkOutDate" bind:value={checkOutDate} />
<LocalStorage key="checkInDate" bind:value={checkInDate} />

<Header platformName="Fleetwächter" expandedByDefault={false} expansionBreakpoint={320}>
  <svelte:fragment slot="skip-to-content">
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
  </HeaderNav>
</Header>

<Content>
  <Grid>
    <Row>
      <Column>
        <NumberInput hideSteppers label="Current kilometers" bind:value={currentKm} />
        <Accordion>
          <AccordionItem title="Configure kilometer allowance">
            <Grid>
              <Row>
                <Column>
                  <DatePicker datePickerType="single" dateFormat="d/m/Y" on:change>
                    <DatePickerInput
                      labelText="Check out date (when you got the car)"
                      bind:value={checkOutDate}
                    />
                  </DatePicker>
                </Column>
                <Column>
                  <DatePicker datePickerType="single" dateFormat="d/m/Y" on:change>
                    <DatePickerInput
                      labelText="Check in date (when you return the car)"
                      bind:value={checkInDate}
                    />
                  </DatePicker>
                </Column>
              </Row>
              <Row>
                <Column>
                  <NumberInput
                    hideSteppers
                    label="Monthly kilometer allowance"
                    bind:value={monthlyKmAllowance}
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
            </Grid>
          </AccordionItem>
        </Accordion>
      </Column>
    </Row>
  </Grid>
</Content>
