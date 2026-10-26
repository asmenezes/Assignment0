<script lang="ts">
    import type { TMovie } from "../types";
    import * as d3 from "d3";
    import { onMount, tick } from "svelte";
    // Keep your other imports and code.
    let brushElement: SVGGElement;

type TProps = {
    movies: TMovie[];
    xCategory: keyof TMovie;
    yCategory: keyof TMovie;
    width?: number;
    height?: number;
    onSelectionChange?: (selectedMovies: TMovie[]) => void;
};

const {
    movies,
    xCategory,
    yCategory,
    width = 800,
    height = 500,
    onSelectionChange,
}: TProps = $props();


    const margin = { top: 45, right: 30, bottom: 90, left: 100 };

    const usableArea = $derived({
        top: margin.top,
        right: width - margin.right,
        bottom: height - margin.bottom,
        left: margin.left,
    });

function categoryValues(movie: TMovie, category: keyof TMovie): string[] {
    const value = movie[category];

    if (category === "year" && value instanceof Date) {
        return [String(value.getFullYear())];
    }

    if (Array.isArray(value)) {
        return value.map((item) => String(item ?? ""));
    }

    if (category === "genres" && typeof value === "string") {
        return value
            .split(",")
            .map((genre) => genre.trim())
            .filter(Boolean);
    }

    return [String(value ?? "")];
}


const xCategories = $derived(
    [...new Set(movies.flatMap((movie) => categoryValues(movie, xCategory)))].sort(),
);

const yCategories = $derived(
    [...new Set(movies.flatMap((movie) => categoryValues(movie, yCategory)))].sort(),
);

const cellCounts = $derived.by(() => {
    const counts = new Map<string, number>();

    for (const movie of movies) {
        const xValues = categoryValues(movie, xCategory);
        const yValues = categoryValues(movie, yCategory);

        for (const xValue of xValues) {
            for (const yValue of yValues) {
                const key = JSON.stringify([xValue, yValue]);
                counts.set(key, (counts.get(key) ?? 0) + 1);
            }
        }
    }

    return counts;
});



    const maxCount = $derived(
        d3.max([...cellCounts.values()]) ?? 0,
    );

    const xScale = $derived(
        d3
            .scaleBand<string>()
            .domain(xCategories)
            .range([usableArea.left, usableArea.right])
            .padding(0.05),
    );

    const yScale = $derived(
        d3
            .scaleBand<string>()
            .domain(yCategories)
            .range([usableArea.top, usableArea.bottom])
            .padding(0.05),
    );

    const colorScale = $derived(
        d3
            .scaleSequential(d3.interpolateGreys)
            .domain([0, Math.max(1, maxCount)]),
    );

    function countFor(xValue: string, yValue: string): number {
        return cellCounts.get(JSON.stringify([xValue, yValue])) ?? 0;
    }

    function cellColor(count: number): string {
        // Keep empty cells white; positive counts use a grayscale intensity.
        return count === 0 ? "#fff" : colorScale(count);
    }
let brushBehavior: d3.BrushBehavior<unknown> | undefined;

onMount(() => {
    brushBehavior = d3
        .brush()
        .extent([
            [usableArea.left, usableArea.top],
            [usableArea.right, usableArea.bottom],
        ])
        .on("end", (event: d3.D3BrushEvent<unknown>) => {
            if (!event.selection) {
                onSelectionChange?.([]);
                return;
            }

            const [[x0, y0], [x1, y1]] = event.selection;

            // Find the cells intersected by the brushed rectangle.
            const selectedCells = new Set<string>();

            for (const xValue of xCategories) {
                for (const yValue of yCategories) {
                    const cellX = xScale(xValue)!;
                    const cellY = yScale(yValue)!;
                    const cellRight = cellX + xScale.bandwidth();
                    const cellBottom = cellY + yScale.bandwidth();

                    const intersects =
                        cellX <= x1 &&
                        cellRight >= x0 &&
                        cellY <= y1 &&
                        cellBottom >= y0;

                    if (intersects) {
                        selectedCells.add(JSON.stringify([xValue, yValue]));
                    }
                }
            }

            const selectedMovies = movies.filter((movie) => {
                const xValues = categoryValues(movie, xCategory);
                const yValues = categoryValues(movie, yCategory);

                return xValues.some((xValue) =>
                    yValues.some((yValue) =>
                        selectedCells.has(JSON.stringify([xValue, yValue])),
                    ),
                );
            });

            onSelectionChange?.(selectedMovies);
        });

    d3.select(brushElement).call(brushBehavior);

    return () => {
        d3.select(brushElement).on(".brush", null);
    };
});

// Keep the brush extent in sync when the chart size or margins change.
$effect(() => {
    if (!brushBehavior || !brushElement) return;

    brushBehavior.extent([
        [usableArea.left, usableArea.top],
        [usableArea.right, usableArea.bottom],
    ]);

    void tick().then(() => {
        d3.select(brushElement).call(brushBehavior!);
    });
});


</script>

<svg {width} {height}>
    <g class="cells">
        {#each yCategories as yValue}
            {#each xCategories as xValue}
                {@const count = countFor(xValue, yValue)}
                <g>
                    <rect
                        x={xScale(xValue)}
                        y={yScale(yValue)}
                        width={xScale.bandwidth()}
                        height={yScale.bandwidth()}
                        fill={cellColor(count)}
                        stroke="#ddd"
                    />

                </g>
            {/each}
        {/each}
    </g>



<!-- Place this after the cells, so the brush sits above them. -->
<g bind:this={brushElement} class="brush"></g>

    {#each xCategories as value}
        <text
            x={(xScale(value) ?? 0) + xScale.bandwidth() / 2}
            y={usableArea.bottom + 15}
            text-anchor="end"
            transform="rotate(-45, {(xScale(value) ?? 0) + xScale.bandwidth() / 2}, {usableArea.bottom + 15})"
            class="axis-label"
        >
            {value}
        </text>
    {/each}

    {#each yCategories as value}
        <text
            x={usableArea.left - 10}
            y={(yScale(value) ?? 0) + yScale.bandwidth() / 2}
            text-anchor="end"
            dominant-baseline="middle"
            class="axis-label"
        >
            {value}
        </text>
    {/each}

    <text
        x={(usableArea.left + usableArea.right) / 2}
        y={height - 10}
        text-anchor="middle"
        class="axis-title"
    >
        {String(xCategory)}
    </text>

    <text
        x={15}
        y={(usableArea.top + usableArea.bottom) / 2}
        text-anchor="middle"
        transform="rotate(-90, 15, {(usableArea.top + usableArea.bottom) / 2})"
        class="axis-title"
    >
        {String(yCategory)}
    </text>
</svg>

<style>
    .cells rect {
        shape-rendering: crispEdges;
    }

    .axis-label {
        font-size: 11px;
    }

    .axis-title {
        font-size: 12px;
        font-weight: 600;
    }
</style>
