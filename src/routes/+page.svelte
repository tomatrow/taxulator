<script module lang="ts">
	import { type Action } from "svelte/action"
	import { on } from "svelte/events"
	import "normalize.css/normalize.css"
	import "magick.css/magick.css"

	type Status =
		| "single"
		| "married_filing_jointly"
		| "married_filing_separately"
		| "head_of_household"

	interface Bracket {
		taxRate: number
		single: number
		married_filing_jointly: number
		married_filing_separately: number
		head_of_household: number
	}

	function currencyFormat(price: number) {
		return price.toLocaleString("en-US", {
			style: "currency",
			currency: "USD",
			currencyDisplay: "symbol"
		})
	}

	const dualInputFormat: Action<
		HTMLInputElement,
		{ parseText?(value: string): number; formatNumber?(value: number): string } | undefined
	> = function dualInputFormat(
		element,
		{ parseText = parseFloat, formatNumber = (value: number) => String(value) } = {}
	) {
		let value = parseText(element.value)

		function beginTextMode() {
			value = parseText(element.value)
			element.type = "text"
			element.value = formatNumber(value)
		}

		function beginNumberMode() {
			element.type = "number"
			// @ts-expect-error
			element.value = value
		}

		const offFocus = on(element, "focus", beginNumberMode)
		const offBlur = on(element, "blur", beginTextMode)

		beginTextMode()

		return {
			destroy() {
				offFocus()
				offBlur()
			}
		}
	}

	const federalTaxBrackets: Bracket[] = [
		{
			taxRate: 0.1,
			single: 11600,
			married_filing_jointly: 23200,
			married_filing_separately: 11600,
			head_of_household: 16550
		},
		{
			taxRate: 0.12,
			single: 47150,
			married_filing_jointly: 94300,
			married_filing_separately: 47150,
			head_of_household: 63100
		},
		{
			taxRate: 0.22,
			single: 100525,
			married_filing_jointly: 201050,
			married_filing_separately: 100525,
			head_of_household: 100500
		},
		{
			taxRate: 0.24,
			single: 191950,
			married_filing_jointly: 383900,
			married_filing_separately: 191950,
			head_of_household: 191950
		},
		{
			taxRate: 0.32,
			single: 243725,
			married_filing_jointly: 487450,
			married_filing_separately: 243725,
			head_of_household: 243700
		},
		{
			taxRate: 0.35,
			single: 609350,
			married_filing_jointly: 731200,
			married_filing_separately: 365600,
			head_of_household: 609350
		}
	]

	const californiaTaxBrackets: Bracket[] = [
		{
			taxRate: 0.01,
			single: 10412,
			married_filing_jointly: 20824,
			married_filing_separately: 10412,
			head_of_household: 20839
		},
		{
			taxRate: 0.02,
			single: 24684,
			married_filing_jointly: 49368,
			married_filing_separately: 24684,
			head_of_household: 49371
		},
		{
			taxRate: 0.04,
			single: 38959,
			married_filing_jointly: 77918,
			married_filing_separately: 38959,
			head_of_household: 63644
		},
		{
			taxRate: 0.06,
			single: 54081,
			married_filing_jointly: 108162,
			married_filing_separately: 54081,
			head_of_household: 78765
		},
		{
			taxRate: 0.08,
			single: 68350,
			married_filing_jointly: 136700,
			married_filing_separately: 68350,
			head_of_household: 93037
		},
		{
			taxRate: 0.093,
			single: 349137,
			married_filing_jointly: 698274,
			married_filing_separately: 349137,
			head_of_household: 474824
		},
		{
			taxRate: 0.103,
			single: 418961,
			married_filing_jointly: 837922,
			married_filing_separately: 418961,
			head_of_household: 569790
		},
		{
			taxRate: 0.113,
			single: 698271,
			married_filing_jointly: 1396542,
			married_filing_separately: 698271,
			head_of_household: 949649
		}
	]

	function getIncomeTax(
		taxableIncome: number,
		brackets: {
			limit: number
			taxRate: number
		}[]
	) {
		return brackets
			.map((bracket, index) => {
				const previousBracket = brackets[index - 1]
				const min = previousBracket?.limit ?? 0
				const max = Math.min(bracket.limit, taxableIncome)
				return Math.max((max - min) * bracket.taxRate, 0)
			})
			.reduce((a, b) => a + b, 0)
	}

	function getFederalTax(
		income: number,
		status: Status,
		isSelfEmployed: boolean,
		qualifiedBusinessIncomePercent: number
	) {
		const federalSelfEmploymentTaxRate = 0.153
		const standardDeduction =
			status === "single" || status === "married_filing_separately" ? 14600 : 27700

		const selfEmploymentTax = isSelfEmployed ? income * federalSelfEmploymentTaxRate : 0
		const selfEmploymentTaxDeduction =
			isSelfEmployed ? selfEmploymentTax / 2 + qualifiedBusinessIncomePercent * income : 0

		const taxableIncome = income - selfEmploymentTaxDeduction - standardDeduction
		const incomeTax = getIncomeTax(
			taxableIncome,
			federalTaxBrackets.map(({ taxRate, [status]: limit }) => ({ taxRate, limit }))
		)

		return { incomeTax, selfEmploymentTax, standardDeduction }
	}

	function getCaliforniaTax(income: number, status: Status) {
		const standardDeduction =
			status === "single" || status === "married_filing_separately" ? 5363 : 10726
		const taxableIncome = income - standardDeduction
		const incomeTax = getIncomeTax(
			taxableIncome,
			californiaTaxBrackets.map(({ taxRate, [status]: limit }) => ({ taxRate, limit }))
		)

		return { incomeTax, taxableIncome, standardDeduction }
	}
</script>

<script lang="ts">
	let income = $state(100000)
	let status: Status = $state("single")
	let isSelfEmployed = $state(true)
	let qualifiedBusinessIncomePercent = $state(0)

	const california = $derived(getCaliforniaTax(income, status))
	const federal = $derived(
		getFederalTax(income, status, isSelfEmployed, qualifiedBusinessIncomePercent / 100)
	)
	const tax = $derived(california.incomeTax + federal.incomeTax + federal.selfEmploymentTax)
	const effectiveTaxRate = $derived(income && tax / income)
	const takeHomeIncome = $derived(income - tax)
	const runRate = $derived(takeHomeIncome / 12)
</script>

<header>
	<h1><ruby>Taxulator<rt>( The California )</rt></ruby></h1>
</header>

<section>
	<h3>Info</h3>
	<form>
		<label for="income">Income</label>
		<input
			id="income"
			type="number"
			min={0}
			bind:value={income}
			use:dualInputFormat={{
				parseText(value) {
					const parsedValue = parseFloat(value)
					if (isNaN(parsedValue)) return 0
					return Math.max(0, parsedValue)
				},
				formatNumber: currencyFormat
			}}
		/>

		<label for="status">Status</label>
		<select id="status" bind:value={status}>
			<option value="single">Single</option>
			<option value="married_filing_jointly">Married Filing Jointly</option>
			<option value="married_filing_separately">Married Filing Separately</option>
			<option value="head_of_household">Head of Household</option>
		</select>

		<fieldset>
			<input id="is_self_employed" type="checkbox" bind:checked={isSelfEmployed} />
			<label for="is_self_employed">Self Employed</label>
		</fieldset>

		{#if isSelfEmployed}
			<label for="qualified_business_income_percent">Qualified Business Income Percent</label>
			<input
				id="qualified_business_income_percent"
				type="number"
				min={0}
				max={20}
				bind:value={qualifiedBusinessIncomePercent}
				use:dualInputFormat={{ formatNumber: value => `${value.toFixed(2)}%` }}
			/>
		{/if}
	</form>
</section>

<section>
	<h3>Taxes</h3>

	<p>California Income Tax: {currencyFormat(california.incomeTax)}</p>
	<p>Federal Income Tax: {currencyFormat(federal.incomeTax)}</p>

	{#if isSelfEmployed}
		<p>Federal Self Employment Tax: {currencyFormat(federal.selfEmploymentTax)}</p>
	{/if}
</section>

<section>
	<h3>Summary</h3>
	<p>Tax: {currencyFormat(tax)}</p>
	<p>Effective Tax Rate: {(effectiveTaxRate * 100).toFixed(2)}%</p>
	<p>Take Home: {currencyFormat(takeHomeIncome)}</p>
	<p>Run Rate: {currencyFormat(runRate)}/month</p>
</section>

<footer>Praise Be</footer>
