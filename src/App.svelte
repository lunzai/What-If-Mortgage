<script>
    import { chart } from "svelte-apexcharts";

    let balance = $state(850000);
    let annualRate = $state(3.5);
    let monthlyPayment = $state(3500);

    let compare1 = $state(4000);
    let compare2 = $state(5000);
    let compare3 = $state(7000);

    let additionalPayment1 = $state(0);
    let additionalPaymentMonth1 = $state(11); // December (0-indexed)
    let additionalPayment2 = $state(0);
    let additionalPaymentMonth2 = $state(11);
    let additionalPayment3 = $state(0);
    let additionalPaymentMonth3 = $state(11);

    function computeSchedule(principal, annualPct, payment, additionalPayment = 0, additionalPaymentMonth = 11) {
        const monthlyRate = annualPct / 100 / 12;
        let balanceLeft = principal;
        let months = 0;
        let totalInterest = 0;
        const timeline = [];

        while (balanceLeft > 0 && months < 1000 * 12) {
            const interest = balanceLeft * monthlyRate;
            let principalPaid = payment - interest;
            if (principalPaid <= 0) {
                return { months: Infinity, totalInterest: Infinity, timeline };
            }
            if (principalPaid > balanceLeft) {
                principalPaid = balanceLeft;
            }
            balanceLeft = +(balanceLeft - principalPaid).toFixed(2);
            totalInterest += interest;
            months += 1;

            // Apply additional payment if it's the specified month of the year
            const currentMonth = (months - 1) % 12; // 0-indexed month in the current year
            if (additionalPayment > 0 && currentMonth === additionalPaymentMonth && balanceLeft > 0) {
                const additionalPrincipal = Math.min(additionalPayment, balanceLeft);
                balanceLeft = +(balanceLeft - additionalPrincipal).toFixed(2);
            }

            if (months % 12 === 0 || balanceLeft === 0) {
                const years = Math.floor(months / 12);
                timeline.push({
                    year: years,
                    balance: Math.max(0, +balanceLeft.toFixed(2)),
                    cumInterest: +totalInterest.toFixed(2),
                    months,
                });
            }
        }

        return { months, totalInterest: +totalInterest.toFixed(2), timeline };
    }

    function fmtMoney(n) {
        return n === Infinity
            ? "—"
            : n.toLocaleString(undefined, { maximumFractionDigits: 0 });
    }

    function fmtYearsMonths(months) {
        if (!isFinite(months)) return "Never (payment too small)";
        const y = Math.floor(months / 12);
        const m = months % 12;
        return `${y}y ${m}m`;
    }

    const baseline = $derived(
        computeSchedule(balance, annualRate, monthlyPayment),
    );
    const cmp1 = $derived(computeSchedule(balance, annualRate, compare1, additionalPayment1, additionalPaymentMonth1));
    const cmp2 = $derived(computeSchedule(balance, annualRate, compare2, additionalPayment2, additionalPaymentMonth2));
    const cmp3 = $derived(computeSchedule(balance, annualRate, compare3, additionalPayment3, additionalPaymentMonth3));

    const chartOptions = $derived({
        chart: { 
            type: "line", 
            height: 400, 
            toolbar: { show: false },
            zoom: { enabled: false }
        },
        stroke: { width: 3 },
        xaxis: { title: { text: "Years" } },
        yaxis: { title: { text: "Remaining Balance" } },
        tooltip: { 
            shared: true,
            y: { formatter: (val) => val.toLocaleString() },
            x: { formatter: (val) => 'Year: ' + val } 
        },
        series: [
            {
                name: "Baseline",
                data: baseline.timeline.map((d) => { return { x: d.year, y: d.balance } }),
            },
            {
                name: "Compare 1",
                data: cmp1.timeline.map((d) => { return { x: d.year, y: d.balance } }),
            },
            {
                name: "Compare 2",
                data: cmp2.timeline.map((d) => { return { x: d.year, y: d.balance } }),
            },
            {
                name: "Compare 3",
                data: cmp3.timeline.map((d) => { return { x: d.year, y: d.balance } }),
            },
        ],
    });

    function donutChartOptions(compare) {
        const total = balance + compare.totalInterest;
        const principalPercent = balance / total * 100;
        const interestPercent = 100 - principalPercent;
        return {
            type: 'donut',
            series: [principalPercent.toFixed(2), interestPercent.toFixed(2)],
            labels: ['Principal', 'Interest'],
            dataLabels: {
                formatter: (val) => {
                    return val + '%';
                }
            }
        }
    }
</script>

{#snippet comparePercent(compare, baseline, positiveColor = 'green', negativeColor = 'red')}
    <span class="{compare > baseline ? `text-${positiveColor}-500` : `text-${negativeColor}-500`}">
        ({compare > baseline ? '+' : ''}{((compare - baseline) / baseline * 100).toFixed(2)}%)
    </span>                                
{/snippet}

{#snippet compareMonths(compare, baseline)}
    <span class="{compare > baseline ? 'text-red-500' : 'text-green-500'}">
        ({compare > baseline ? '+' : '-'}{fmtYearsMonths(Math.abs(baseline - compare))})
    </span>
{/snippet}

<div class="max-w-5xl mx-auto p-6">
    <div class="bg-white rounded-2xl shadow p-8">
        <h1 class="text-4xl font-semibold mb-4">What If Mortgage</h1>
        <p class="text-sm text-slate-500 mb-6">
            Set baseline and compare repayments to see payoff time and interest
            savings.
        </p>

        <div class="grid grid-cols-1 md:grid-cols-3 gap-6 mb-6">
            <label class="block">
                <div class="text-xs text-slate-600">Current Balance</div>
                <input
                    type="number"
                    bind:value={balance}
                    class="mt-2 w-full p-2 border rounded"
                />
            </label>

            <label class="block">
                <div class="text-xs text-slate-600">
                    Annual Interest Rate (%)
                </div>
                <input
                    type="number"
                    step="0.01"
                    bind:value={annualRate}
                    class="mt-2 w-full p-2 border rounded"
                />
            </label>

            <label class="block">
                <div class="text-xs text-slate-600">Baseline Payment</div>
                <input
                    type="number"
                    bind:value={monthlyPayment}
                    class="mt-2 w-full p-2 border rounded"
                />
            </label>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-3 gap-6 mb-6">
            <label class="block">
                <div class="text-xs text-slate-600">Compare Payment 1</div>
                <input
                    type="number"
                    bind:value={compare1}
                    class="mt-2 w-full p-2 border rounded"
                />
            </label>

            <label class="block">
                <div class="text-xs text-slate-600">Compare Payment 2</div>
                <input
                    type="number"
                    bind:value={compare2}
                    class="mt-2 w-full p-2 border rounded"
                />
            </label>

            <label class="block">
                <div class="text-xs text-slate-600">Compare Payment 3</div>
                <input
                    type="number"
                    bind:value={compare3}
                    class="mt-2 w-full p-2 border rounded"
                />
            </label>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-3 gap-6 mb-10">
            <div class="space-y-4">
                <label class="block">
                    <div class="text-xs text-slate-600">Additional Payment 1</div>
                    <input
                        type="number"
                        bind:value={additionalPayment1}
                        class="mt-2 w-full p-2 border rounded"
                        placeholder="0"
                    />
                </label>
                <label class="block">
                    <div class="text-xs text-slate-600">Payment Month 1</div>
                    <select
                        bind:value={additionalPaymentMonth1}
                        class="mt-2 w-full p-2 border rounded"
                    >
                        <option value={0}>January</option>
                        <option value={1}>February</option>
                        <option value={2}>March</option>
                        <option value={3}>April</option>
                        <option value={4}>May</option>
                        <option value={5}>June</option>
                        <option value={6}>July</option>
                        <option value={7}>August</option>
                        <option value={8}>September</option>
                        <option value={9}>October</option>
                        <option value={10}>November</option>
                        <option value={11}>December</option>
                    </select>
                </label>
            </div>

            <div class="space-y-4">
                <label class="block">
                    <div class="text-xs text-slate-600">Additional Payment 2</div>
                    <input
                        type="number"
                        bind:value={additionalPayment2}
                        class="mt-2 w-full p-2 border rounded"
                        placeholder="0"
                    />
                </label>
                <label class="block">
                    <div class="text-xs text-slate-600">Payment Month 2</div>
                    <select
                        bind:value={additionalPaymentMonth2}
                        class="mt-2 w-full p-2 border rounded"
                    >
                        <option value={0}>January</option>
                        <option value={1}>February</option>
                        <option value={2}>March</option>
                        <option value={3}>April</option>
                        <option value={4}>May</option>
                        <option value={5}>June</option>
                        <option value={6}>July</option>
                        <option value={7}>August</option>
                        <option value={8}>September</option>
                        <option value={9}>October</option>
                        <option value={10}>November</option>
                        <option value={11}>December</option>
                    </select>
                </label>
            </div>

            <div class="space-y-4">
                <label class="block">
                    <div class="text-xs text-slate-600">Additional Payment 3</div>
                    <input
                        type="number"
                        bind:value={additionalPayment3}
                        class="mt-2 w-full p-2 border rounded"
                        placeholder="0"
                    />
                </label>
                <label class="block">
                    <div class="text-xs text-slate-600">Payment Month 3</div>
                    <select
                        bind:value={additionalPaymentMonth3}
                        class="mt-2 w-full p-2 border rounded"
                    >
                        <option value={0}>January</option>
                        <option value={1}>February</option>
                        <option value={2}>March</option>
                        <option value={3}>April</option>
                        <option value={4}>May</option>
                        <option value={5}>June</option>
                        <option value={6}>July</option>
                        <option value={7}>August</option>
                        <option value={8}>September</option>
                        <option value={9}>October</option>
                        <option value={10}>November</option>
                        <option value={11}>December</option>
                    </select>
                </label>
            </div>
        </div>

        <div class="mb-10">
            <h3 class="text-xl font-semibold mb-4">Balance Over Time</h3>
            <div use:chart={chartOptions}></div>
        </div>

        {#snippet pieChart(title, balance, interest)}
            <div use:chart={{
                chart: {
                    type: 'donut',
                    width: 250,
                },
                series: [Number(balance.toFixed(2)), Number(interest.toFixed(2))],
                labels: ['Principal', 'Interest'],
                legend: {
                    position: 'bottom',
                },
                title: {
                    text: title
                },
            }}></div>
        {/snippet}

        <div class="mb-10">
            <h3 class="text-xl font-semibold mb-4">Loan Distribution</h3>
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
                {@render pieChart('Baseline', balance, baseline.totalInterest)}
                {@render pieChart('Compare 1', balance, cmp1.totalInterest)}
                {@render pieChart('Compare 2', balance, cmp2.totalInterest)}
                {@render pieChart('Compare 3', balance, cmp3.totalInterest)}
            </div>
        </div>

        <div class="mb-10">
            <h3 class="font-semibold mb-4 text-xl">Comparison Table</h3>
            <div class="overflow-x-auto">
                <table class="text-sm w-full border border-slate-200 rounded table-auto">
                    <thead class="bg-slate-100">
                        <tr>
                            <th class="p-3 text-left">Scenario</th>
                            <th class="p-3 text-left">Payment</th>
                            <th class="p-3 text-left">Payoff Time</th>
                            <th class="p-3 text-left">Total Interest</th>
                            <th class="p-3 text-left">Total Repayment</th>
                            <th class="p-3 text-left">Interest Saved</th>
                            <th class="p-3 text-left">Months Saved</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr class="border-t">
                            <td class="p-3">Baseline</td>
                            <td class="p-3 text-left">
                                {monthlyPayment.toLocaleString()}
                            </td>
                            <td class="p-3 text-left">{fmtYearsMonths(baseline.months)}</td>
                            <td class="p-3 text-left">{fmtMoney(baseline.totalInterest)}</td>
                            <td class="p-3 text-left">{fmtMoney(balance + baseline.totalInterest)}</td>
                            <td class="p-3 text-left">—</td>
                            <td class="p-3 text-left">—</td>
                        </tr>
                        {#snippet compareRow(name, value, compareValue, current, previous)}
                            <tr class="border-t">
                                <td class="p-3">{name}</td>
                                <td class="p-3 text-left">
                                    <div class="flex items-center gap-2">
                                        {value.toLocaleString()}
                                        <div class="flex flex-col text-[0.6rem]">
                                            {@render comparePercent(value, monthlyPayment, 'red', 'green')}
                                            {@render comparePercent(value, compareValue, 'red', 'green')}
                                        </div>
                                    </div>
                                </td>
                                <td class="p-3 text-left">
                                    <div class="flex items-center gap-2">
                                            {fmtYearsMonths(current.months)}
                                        <div class="flex flex-col text-[0.6rem]">
                                            {@render compareMonths(current.months, baseline.months)}
                                            {@render compareMonths(current.months, previous.months)}
                                        </div>
                                    </div>
                                </td>
                                <td class="p-3 text-left">
                                    <div class="flex items-center gap-2">
                                        {fmtMoney(current.totalInterest)}
                                        <div class="flex flex-col text-[0.6rem]">
                                            {@render comparePercent(current.totalInterest, baseline.totalInterest, 'red', 'green')}
                                            {@render comparePercent(current.totalInterest, previous.totalInterest, 'red', 'green')}
                                        </div>
                                    </div>
                                </td>
                                <td class="p-3 text-left">
                                    <div class="flex items-center gap-2">
                                        {fmtMoney(balance + current.totalInterest)}
                                        <div class="flex flex-col text-[0.6rem]">
                                            {@render comparePercent(balance + current.totalInterest, balance + baseline.totalInterest, 'red', 'green')}
                                            {@render comparePercent(balance + current.totalInterest, balance + previous.totalInterest, 'red', 'green')}
                                        </div>
                                    </div>
                                </td>
                                <td class="p-3 text-left">{fmtMoney(baseline.totalInterest - current.totalInterest)}</td>
                                <td class="p-3 text-left">{fmtMoney(baseline.months - current.months)}</td>
                            </tr>
                        {/snippet}
                        {@render compareRow('Compare 1', compare1, monthlyPayment, cmp1, baseline)}
                        {@render compareRow('Compare 2', compare2, compare1, cmp2, cmp1)}
                        {@render compareRow('Compare 3', compare3, compare2, cmp3, cmp2)}
                    </tbody>
                </table>
            </div>
        </div>

        <div class="text-sm text-slate-600">
            Tip: Adjust baseline and comparison payments to find the repayment
            plan that balances cashflow and debt freedom.
        </div>
    </div>

    <footer class="text-xs text-slate-400 mt-6 flex justify-between">
        <div class="flex flex-col gap-1">
            <div>Built for quick planning — numbers are approximate and ignore past progressive releases or moratorium distortions.</div>
            <div>
                <a href="https://www.flaticon.com/free-icons/mortgage-loan" target="_blank" title="mortgage loan icons">
                    Mortgage loan icons created by Freepik - Flaticon.
                </a>
            </div>
            <div>
                Built By <a href="https://github.com/lunzai" target="_blank" class="hover:text-indigo-500 transition-colors duration-300" aria-label="View source code on GitHub">lunzai</a>.
            </div>
        </div>
        <div class="flex items-center gap-2">
            <a href="https://github.com/lunzai/What-If-Mortgage" target="_blank" class="social-link hover:text-indigo-500 transition-colors duration-300" aria-label="View source code on GitHub">
                <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M9 19c-5 1.5-5-2.5-7-3m14 6v-3.87a3.37 3.37 0 0 0-.94-2.61c3.14-.35 6.44-1.54 6.44-7A5.44 5.44 0 0 0 20 4.77 5.07 5.07 0 0 0 19.91 1S18.73.65 16 2.48a13.38 13.38 0 0 0-7 0C6.27.65 5.09 1 5.09 1A5.07 5.07 0 0 0 5 4.77a5.44 5.44 0 0 0-1.5 3.78c0 5.42 3.3 6.61 6.44 7A3.37 3.37 0 0 0 9 18.13V22"></path>
                </svg>
            </a>
        </div>
    </footer>
</div>
