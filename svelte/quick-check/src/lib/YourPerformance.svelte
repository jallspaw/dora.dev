<script>
    //@ts-nocheck
    import { onMount } from "svelte";

    import PerformanceGraph from "./PerformanceGraph.svelte";
    import { recode_numeric_range, sendAnalyticsEvent } from "./utils.js";
    import metrics_question_responses from "./data/metrics_question_responses.json";
    import organization_size_metrics from "./data/organization_size_metrics.json";
    import industry_metrics_data from "./data/industry_metrics.json";

    export let metrics, industry, displayMode;

    let metrics_recoded = {
        leadtime: -1,
        deployfreq: -1,
        changefailure: -1,
        failurerecovery: -1,
    };
    let performance_average = 0;
    let industry_metrics = industry_metrics_data; // Default to industry_metrics
    let comparisonType = "industry"; // Default comparison type
    let currentIndustry = industry; // Track the current industry value

    const calculate_recoded_metrics = () => {
        // inputs for these metrics range from 1 to 6; recode to a 0-10 scale
        metrics_recoded.leadtime = recode_numeric_range(
            parseInt(metrics.leadtime),
            1, // input_min
            6, // input_max
            0, // output_min
            10, // output_max
        );
        metrics_recoded.deployfreq = recode_numeric_range(
            parseInt(metrics.deployfreq),
            1, // input_min
            6, // input_max
            0, // output_min
            10, // output_max
        );
        metrics_recoded.failurerecovery = recode_numeric_range(
            parseInt(metrics.failurerecovery),
            1, // input_min
            6, // input_max
            0, // output_min
            10, // output_max
        );

        // inputs for change failure range from 0 to 100, and higher is worse; recode to a 10-0 scale
        metrics_recoded.changefailure = parseFloat(
            recode_numeric_range(
                parseInt(metrics.changefailure),
                0, // input_min
                100, // input_max
                10, // output_min
                0, // output_max
            ),
        );
    };

    const setIndustryInURL = (industry) => {
        if (typeof window !== "undefined") {
            const url = new URL(window.location);
            url.searchParams.set("industry", industry);
            window.history.replaceState({}, "", url);
        }
    };

    onMount(() => {
        window.scrollTo({
            top: document.getElementById("results").offsetTop,
            behavior: "smooth",
        });

        sendAnalyticsEvent("quick_check_results");

         // Check for "comp" in query string on mount
         if (typeof window !== "undefined") {
            const urlParams = new URLSearchParams(window.location.search);
            const comp = urlParams.get("comp");
            if (comp === "size") {
                industry_metrics = organization_size_metrics;
                comparisonType = "size";
            } else {
                industry_metrics = industry_metrics_data;
                comparisonType = "industry";
            }
        }
    });

    $: metrics, calculate_recoded_metrics();
    $: performance_average = (
        (metrics_recoded.leadtime +
            metrics_recoded.deployfreq +
            metrics_recoded.changefailure +
            metrics_recoded.failurerecovery) /
        4
    ).toFixed(1);
    $: {
        if (!industry_metrics[industry]) {
            currentIndustry = "all"; // Reset to "all" if not found
            console.warn(`Industry "${industry}" not found in ${comparisonType} dataset. Resetting to "all".`);
            industry = "all";
            setIndustryInURL(currentIndustry);
        } else {
            currentIndustry = industry;
        }
    }
    $: selected_industry_metrics = industry_metrics[currentIndustry];
    $: setIndustryInURL(currentIndustry);
    $: comparisonText = comparisonType === "industry" ? "Compare to industry benchmark:" : "Compare to organization size benchmark:";
    $: baselineText = comparisonType === "industry" ? `2024 Industry baseline (${industry_metrics[industry]["name"]}):` : `2024 Organization size benchmark (${industry_metrics[currentIndustry]["name"]}):`;
</script>

<div class="heading">
    <h1 id="results">Your software delivery performance</h1>
    {comparisonText}
    <select bind:value={industry}>
        {#each Object.entries(industry_metrics) as [industry, industry_data]}
            <option value={industry}>{industry_data["name"]}</option>
        {/each}
    </select>
</div>
<div class="YourPerformance {displayMode}">
    <section class="performance-graphs">
        <aside>
            <b>Your performance</b>
            <span
                class="performance-average"
                style:background-position={`${performance_average * 10}%`}
                >{performance_average}</span
            >
        </aside>
        <div class="graph">
            <PerformanceGraph
                user_score={performance_average}
                industry_score={selected_industry_metrics.performance_average
                    .mean}
                std={selected_industry_metrics.performance_average.std}
                tickmarks={[0, 2, 4, 6, 8, 10]}
                featured
                {displayMode}
            />
        </div>
        <aside>
            <b>Lead time</b>
            <span class="metric_description"
                >{metrics_question_responses.leadtime[metrics.leadtime]}</span
            >
        </aside>
        <div class="graph">
            <PerformanceGraph
                user_score={metrics_recoded.leadtime}
                industry_score={selected_industry_metrics.leadtime.mean}
                std={selected_industry_metrics.leadtime.std}
                tickmarks={[">6mo", "1-6mo", "1w-1mo", "1d-1w", "<1d", "<1h"]}
                {displayMode}
            />
        </div>
        <aside>
            <b>Deploy frequency</b>
            <span class="metric_description"
                >{metrics_question_responses.deployfreq[
                    metrics.deployfreq
                ]}</span
            >
        </aside>
        <div class="graph">
            <PerformanceGraph
                user_score={metrics_recoded.deployfreq}
                industry_score={selected_industry_metrics.deployfreq.mean}
                std={selected_industry_metrics.deployfreq.std}
                tickmarks={[
                    "<6mo",
                    "1-6mo",
                    "1w-1mo",
                    "1d-1w",
                    "1h-1d",
                    "on demand",
                ]}
                {displayMode}
            />
        </div>
        <aside>
            <b>Change fail rate</b>
            <span class="metric_description"
                >{metrics.changefailure}% of changes fail</span
            >
        </aside>
        <div class="graph">
            <!-- display change fail score with zero decimal places if it's a round number, else round to 1 decimal place -->
            <PerformanceGraph
                user_score={+metrics_recoded.changefailure.toFixed(1)}
                industry_score={selected_industry_metrics.changefailure.mean}
                std={selected_industry_metrics.changefailure.std}
                tickmarks={[
                    "100%",
                    "90%",
                    "80%",
                    "70%",
                    "60%",
                    "50%",
                    "40%",
                    "30%",
                    "20%",
                    "10%",
                    "0%",
                ]}
                {displayMode}
            />
        </div>
        <aside>
            <b>Failed deployment recovery time</b>
            <span class="metric_description"
                >{metrics_question_responses.failurerecovery[
                    metrics.failurerecovery
                ]}</span
            >
        </aside>
        <div class="graph">
            <PerformanceGraph
                user_score={metrics_recoded.failurerecovery}
                industry_score={selected_industry_metrics.failurerecovery.mean}
                std={selected_industry_metrics.failurerecovery.std}
                tickmarks={[">6mo", "1-6mo", "1w-1mo", "1d-1w", "<1d", "<1h"]}
                {displayMode}
            />
        </div>
    </section>
    <section class="legend">
        <div class="legend-header">
            <span>
            {baselineText}
            </span>
        </div>
        <div class="legend-item">
            <span class="industry">&nbsp;</span> Average
        </div>
        <div class="legend-item">
            <span class="std">&nbsp;</span> Standard deviation
        </div>
        <div class="legend-item">
            <span class="your">&nbsp;</span> Your performance
        </div>
    </section>
</div>

<style lang="scss">
    .heading {
        text-align: center;
    }

    .YourPerformance {
        display: flex;
        flex-direction: column;

        .performance-graphs {
            display: grid;
            align-items: center;
            grid-template-columns: fit-content(20rem) auto;
            gap: 2rem 2rem;
            margin-top: 2rem;
            padding: 0 1.25em;

            aside {
                text-align: center;
                font-size: 0.75rem;
                color: #999;

                b {
                    display: block;
                    font-size: 1rem;
                    color: #333;
                    font-weight:500;
                }

                .performance-average {
                    display: inline-block;
                    background: var(--performance-spectrum);
                    background-size: 100vw;
                    color: white;
                    font-size: 2.75rem;
                    padding: 0.25rem 1.5rem;
                    border-radius: 1rem;
                }
            }
        }
        .legend {
            display: flex;
            flex-direction: column; /* Stack items vertically */
            align-items: flex-start; /* Align items to the start of the container */
            align-items: center; /* Center align items vertically */
            margin-top: 3rem;
            font-size: 0.75rem;
            color: #666;
            justify-content: center;

                .legend-header {
                    margin-right: 1.5rem; /* Add spacing between header and items */
                    margin-bottom: 1rem; /* Add spacing below the header */
                }

                .legend-item {
                    margin-right: 1.5rem; /* Add spacing between legend items */
                    margin-bottom: 0.5rem; /* Add spacing between legend items */
                    display: flex;
                    align-items: center;
                }

                span {
                    display: inline-block;
                    height: 1.5rem;
                    vertical-align: middle;
                    margin-left: 0.5rem;

                    &.your {
                        width: 4px;
                        height: 1rem;
                        background-color: var(--dora-blue);
                        border-radius: 2px;
                        margin-right: 0.5rem;
                    }

                    &.industry {
                        background-color: var(--metric-background) !important;
                        width: 1px;
                        height: 1rem;
                        margin-right: 0.5rem;
                    }

                    &.std {
                        background-color: var(--std-background);
                        width: 32px;
                        height: 1rem;
                        border-radius: 0.25rem;
                        margin-right: 0.5rem;
                    }
            }
        }

        &.kiosk {
            flex-direction: row;
            .performance-graphs {
                gap: 0.25rem 2rem;
                flex-grow: 1;
            }
            .legend {
                flex-direction: column;
                div {
                    text-align: center;
                    margin-bottom: 1rem;
                }
            }
        }
    }

    /* There's no elegant way to use global variables for media queries (css variables aren't supported for this purpose,
    and SCSS vars are hard to propagate between different svelte components).
    So we'll use a "magic number" of 800px, in each file */
    @media (max-width: 800px) {
        .YourPerformance {
            .performance-graphs {
                grid-template-columns: 1fr;
                padding: 0 2rem;
                gap: 1rem 0;

                aside {
                    b {
                        display: inline-block;
                        margin-right: 1rem;
                    }

                    .metric_description {
                        white-space: nowrap;
                    }
                }

                .graph {
                    border-bottom: 1px dotted var(--border-color-light);
                    padding-bottom: 1.25rem;
                    margin-bottom: 1.25rem;
                }
            }
            .legend {
                flex-direction: column;

                .legend_header {
                    display: block;
                }
            }
        }
    }
</style>
