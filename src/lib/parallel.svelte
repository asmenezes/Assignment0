<script lang="ts">
    import type { TMovie } from "../types";
    import * as d3 from "d3";

    type TProps = {
        movies: TMovie[];
        category: keyof TMovie;
        width?: number;
        height?: number;
    };

    const { movies, category, width = 800, height = 500 }: TProps = $props();

    const margin = { top: 35, right: 30, bottom: 35, left: 30 };
    const usableArea = {
        top: margin.top,
        right: width - margin.right,
        bottom: height - margin.bottom,
        left: margin.left,
    };

    let axes: SVGGElement[] = $state([]);
    function numericValue(
        movie: TMovie,
        attribute: keyof TMovie,
    ): number | undefined {
        const value = movie[attribute];

        if (typeof value === "number") return value;

        if (attribute === "year") {
            if (value instanceof Date) return value.getFullYear();

            const parsed = Number(value);
            return Number.isFinite(parsed) ? parsed : undefined;
        }

        return undefined;
    }

    type PlotRow = {
        movie: TMovie;
        categoryValue: string;
    };

    const plotRows = $derived(
        movies.flatMap((movie) => {
            const value = movie[category];
            const values =
                category === "genres" && Array.isArray(value)
                    ? [...new Set(value)]
                    : [value];

            return values.map((item) => ({
                movie,
                categoryValue: String(item ?? ""),
            }));
        }),
    );

    // Use numeric fields as the quantitative axes.
    const quantitativeAttributes = $derived(
        movies.length
            ? (Object.keys(movies[0]) as (keyof TMovie)[]).filter(
                  (key) =>
                      key !== category &&
                      (key === "year" ||
                          movies.some(
                              (movie) => typeof movie[key] === "number",
                          )),
              )
            : [],
    );

    const categories = $derived([
        ...new Set(plotRows.map((row) => row.categoryValue)),
    ]);

    let brushSelections = $state<
        Partial<Record<keyof TMovie, [number, number]>>
    >({});

    const dimensions = $derived([category, ...quantitativeAttributes]);

    const xScale = $derived(
        d3
            .scalePoint<keyof TMovie>()
            .domain(dimensions)
            .range([usableArea.left, usableArea.right])
            .padding(0.5),
    );

    const categoryScale = $derived(
        d3
            .scaleBand<string>()
            .domain(categories)
            .range([usableArea.bottom, usableArea.top])
            .padding(0.15),
    );

    const numericScales = $derived(
        new Map(
            quantitativeAttributes.map((attribute) => {
                const values = movies
                    .map((movie) => numericValue(movie, attribute))
                    .filter((value): value is number => value !== undefined);

                const isVoteCount = attribute === "num_votes";

                const scale = isVoteCount
                    ? d3
                          .scaleLog()
                          .domain([
                              Math.max(1, d3.min(values) ?? 1),
                              Math.max(1, d3.max(values) ?? 1),
                          ])
                          .range([usableArea.bottom, usableArea.top])
                    : d3
                          .scaleLinear()
                          .domain(d3.extent(values) as [number, number])
                          .nice()
                          .range([usableArea.bottom, usableArea.top]);

                return [attribute, scale];
            }),
        ),
    );

    function yPosition(row: PlotRow, dimension: keyof TMovie): number {
        if (dimension === category) {
            const y = categoryScale(row.categoryValue);
            return y === undefined
                ? (usableArea.top + usableArea.bottom) / 2
                : y + categoryScale.bandwidth() / 2;
        }

        const value = numericValue(row.movie, dimension);
        const scale = numericScales.get(dimension);

        return value !== undefined && scale
            ? scale(dimension === "num_votes" ? Math.max(1, value) : value)
            : (usableArea.top + usableArea.bottom) / 2;
    }

    function isLineSelected(row: PlotRow): boolean {
        return dimensions.every((dimension) => {
            const range = brushSelections[dimension];
            if (!range) return true;

            const y = yPosition(row, dimension);
            return y >= range[0] && y <= range[1];
        });
    }

    function linePath(row: PlotRow): string {
        const points = dimensions.map(
            (dimension) =>
                [xScale(dimension) ?? 0, yPosition(row, dimension)] as [
                    number,
                    number,
                ],
        );

        return d3.line().curve(d3.curveMonotoneX)(points) ?? "";
    }

    function updateAxes() {
        axes.forEach((axis, index) => {
            const dimension = dimensions[index];
            if (!axis || !dimension) return;

            const axisSelection = d3.select(axis);
            axisSelection.selectAll("*").remove();

            if (index === 0) {
                axisSelection.call(
                    d3
                        .axisLeft(
                            d3
                                .scaleBand<string>()
                                .domain(categories)
                                .range([usableArea.bottom, usableArea.top])
                                .padding(0.15),
                        )
                        .tickSize(0),
                );
            } else {
                const scale = numericScales.get(dimension);
                if (scale) axisSelection.call(d3.axisLeft(scale).ticks(5));
            }

            const brush = d3
                .brushY()
                .extent([
                    [-12, usableArea.top],
                    [12, usableArea.bottom],
                ])
                .on("brush end", (event) => {
                    const selection = event.selection as
                        | [number, number]
                        | null;

                    // A null selection means this axis has no active filter.
                    const next = { ...brushSelections };
                    if (selection) {
                        next[dimension] = selection;
                    } else {
                        delete next[dimension];
                    }
                    brushSelections = next;
                });

            axisSelection.append("g").attr("class", "brush").call(brush);
        });
    }

    $effect(() => {
        dimensions;
        categories;
        numericScales;
        updateAxes();
    });
</script>

<svg {width} {height}>
    <g class="lines">
        {#each plotRows as row}
            <path
                d={linePath(row)}
                fill="none"
                stroke={isLineSelected(row) ? "steelblue" : "lightgrey"}
                stroke-width="1.5"
                opacity={isLineSelected(row) ? 0.8 : 0.1}
            />
        {/each}
    </g>

    {#each dimensions as dimension, index}
        <g
            class="axis"
            transform="translate({xScale(dimension)}, 0)"
            bind:this={axes[index]}
        />
        <text
            x={xScale(dimension)}
            y={usableArea.top - 15}
            text-anchor="middle"
            class="axis-label"
        >
            {String(dimension)}
        </text>
    {/each}
</svg>

<style>
    .lines path {
        transition: opacity 0.2s;
    }

    .lines path:hover {
        opacity: 0.9;
        stroke-width: 2.5;
    }

    :global(.axis text) {
        font-size: 11px;
    }

    .axis-label {
        font-size: 12px;
        font-weight: 600;
    }
</style>
