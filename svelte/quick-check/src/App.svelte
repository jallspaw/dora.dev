<script>
    // @ts-nocheck

    import { onMount } from "svelte";
    import MetricsQuestion from "./lib/MetricsQuestion.svelte";
    import YourPerformance from "./lib/YourPerformance.svelte";
    import HelpMePrioritize from "./lib/HelpMePrioritize.svelte";
    import GoFurther from "./lib/GoFurther.svelte";
    import { sendAnalyticsEvent } from "./lib/utils.js";
    import FullScreenButton from "./lib/kiosk/FullScreenButton.svelte";
    import NextSteps from "./lib/kiosk/NextSteps.svelte";
    import StartOver from "./lib/kiosk/StartOver.svelte";

    let metrics = {
        leadtime: -1,
        deployfreq: -1,
        changefailure: -1,
        failurerecovery: -1,
    };

    let step = "input";
    let industry = "all";
    let current_capability = -1;
    let metric_names = Object.keys(metrics);
    let current_metric = 0; // in kiosk mode, metrics questions are presented one at a time
    let displayMode = "embedded";

    function saveURLParams() {
        // in kiosk mode, don't populate URL (b/c user can't easily copy-paste it)
        if (typeof window !== "undefined" && displayMode === "embedded") {
            const url = new URL(window.location);
            metric_names.forEach((metric) =>
                url.searchParams.set(metric, metrics[metric]),
            );
            url.searchParams.set("v", "2024");
            window.history.pushState({}, "", url);
        }
    }

    onMount(() => {
        const searchParams = new URLSearchParams(window.location.search);

        // quick check may be running on a kiosk or tablet, as specified via URL param or in a meta tag from the calling page
        if (searchParams.has("displayMode")) {
            displayMode = searchParams.get("displayMode");
            console.log(
                `displayMode: ${displayMode} provided via querystring parameter`,
            );
        } else if (
            document.getElementsByName("displayMode").length &&
            document.getElementsByName("displayMode")[0].content
        ) {
            displayMode = document.getElementsByName("displayMode")[0].content;
            console.log(`displayMode: ${displayMode} provided via <meta> tag`);
        }

        // if the metric values are passed on the URL, save them to local vars and advance to results
        if (metric_names.every((metric) => searchParams.has(metric))) {
            metric_names.forEach((metric) => {
                metrics[metric] = searchParams.get(metric);
            });
            step = "results";
        }

        // if the capability alues are passed on the URL, advance to priorities
        // !!!This is a hack using hard-coded values. TODO: make more elegant
        if (
            ["ci", "arch", "culture"].every((param) => searchParams.has(param))
        ) {
            current_capability = 2;
        }

        if (searchParams.has("industry")) {
            industry = searchParams.get("industry");
        }

        sendAnalyticsEvent("quick_check_start");

        // TODO: add error handling w/r/t URL params (e.g. if step == "results" but metrics values not present, bounce to input)
    });

    function nextMetric() {
        if (current_metric < 3) {
            current_metric++;
        } else if (displayMode === "kiosk") {
            // in kiosk mode, user automatially advances to results after last answer
            showResults();
        }
    }

    function showResults() {
        saveURLParams();
        step = "results";
    }

    function reset() {
        metric_names.forEach((metric) => {
            metrics[metric] = -1;
        });
        step = "input";
        industry = "all";
        current_capability = -1;
        current_metric = 0;
        saveURLParams();
    }
</script>

<!-- Vite provides environment variables; if running in dev, show some debug -->
{#if typeof import.meta.env.MODE !== "undefined" && import.meta.env.MODE === "development"}
    DisplayMode: {displayMode}<br />
    <label
        ><input
            type="radio"
            name="displayMode"
            value={"embedded"}
            bind:group={displayMode}
        />
        embedded<br /></label
    >
    <label
        ><input
            type="radio"
            name="displayMode"
            bind:group={displayMode}
            value={"kiosk"}
        />
        kiosk<br /></label
    >
{/if}

<!--- END debug -->

{#if displayMode === "kiosk"}
    <FullScreenButton />
{/if}

<div class="quickcheck {displayMode}">
    {#if displayMode === "kiosk"}
        {#if step === "input"}
            <div class="kioskMetricsQuestions">
                <aside>
                    Take the
                    <h1>DORA Quick Check</h1>
                    {#if current_metric > 0}
                        {#key step} <!-- re initialize this widget on every change of step or current_metric -->
                            {#key current_metric}
                                <StartOver on:reset={reset} {displayMode} />
                            {/key}
                        {/key}
                    {/if}
                </aside>
                {#key current_metric}
                    <MetricsQuestion
                        bind:metrics
                        bind:current_metric
                        metric_name={metric_names[current_metric]}
                        metric_position={current_metric}
                        {displayMode}
                        on:nextMetric={nextMetric}
                    />
                {/key}
            </div>
        {:else if step === "results"}
            <div class="yourPerformance">
                <YourPerformance {metrics} bind:industry {displayMode} />
                <NextSteps {displayMode} on:reset={reset} />
            </div>
        {/if}
    {:else}
        {#if step === "input"}
            {#each metric_names as metric, idx}
                <MetricsQuestion
                    bind:metrics
                    metric_name={metric}
                    metric_position={idx}
                    on:nextMetric={nextMetric}
                />
            {/each}
            <section class="submit">
                <button
                    disabled={!metric_names.every(
                        (metric) => metrics[metric] != -1,
                    )}
                    on:click={showResults}>View Results</button
                >
            </section>
        {:else if step === "results" || step === "priorities"}
            <YourPerformance {metrics} bind:industry {displayMode} />
            <HelpMePrioritize bind:current_capability />
        {/if}
        <div class="faq">
            <a href="/faq/#dora-quick-check">Quick Check FAQ</a>
        </div>
        {#if step !== "input"}
            <GoFurther />
        {/if}
    {/if}
</div>

<style lang="scss">
    /* override page-level styles for padding b/c it causes graphs to be mispositioned */
    :global(body div.quickcheck) {
        padding-left: 0;
        padding-right: 0;
        position: relative;
    }

    .faq {
        text-align: center;
        padding-top: 1.5rem;
        font-size: 85%;
    }

    section.submit {
        text-align: center;
    }

    .kioskMetricsQuestions {
        display: flex;
        flex-direction: row;

        aside {
            margin: 0rem 2rem;
            border-right: 1px solid #ccd;
            padding-right: 2rem;
        }

        h1 {
            font-size: 7.5rem;
        }
    }

    aside {
        width: 30%;
        font-size: 2rem;
    }

    .kiosk {
        .yourPerformance {
            margin: 0 2rem 0.5rem 0;
        }
    }
</style>
