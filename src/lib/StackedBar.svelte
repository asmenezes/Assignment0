<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";
  // define the props of the Bar component
  type Props = {
    movies: TMovie[];
    progress?: number;
    width?: number;
    height?: number;
  };
  // progress is 100 by default unless specified
  let { movies, progress = 100, width = 500, height = 400 }: Props = $props();

  let selectedGenre: string | undefined = $state();

  // processing the data; $derived is used to create a reactive variable that updates whenever the dependent variables change
  const yearRange = $derived(d3.extent(movies.map((d) => d.year)));

  function getUpYear(yearRange: [undefined, undefined] | [Date, Date]) {
    if (!yearRange[0]) return new Date();
    const timeScale = d3.scaleTime().domain(yearRange).range([0, 100]);
    return timeScale.invert(progress);
  }
  const upYear: Date = $derived(getUpYear(yearRange!));

  function getGenreNums(
    movies: TMovie[],
    upYear: Date,
  ): { [genre: string]: { [otherGenre: string]: number } } {
    const res: {
      [genre: string]: { [otherGenre: string]: number };
    } = {};

    movies
      .filter((movie) => movie.year <= upYear)
      .forEach((movie) => {
        // Prevent duplicate genres in one movie from being counted multiple times
        const genres = [...new Set(movie.genres)];

        genres.forEach((genre) => {
          if (!res[genre]) {
            res[genre] = {};
          }

          genres.forEach((otherGenre) => {
            if (genre === otherGenre) return;

            res[genre][otherGenre] = (res[genre][otherGenre] || 0) + 1;
          });
        });
      });

    // Sort each genre's related genre counts from highest to lowest
    Object.keys(res).forEach((genre) => {
      res[genre] = Object.fromEntries(
        Object.entries(res[genre]).sort(
          ([, countA], [, countB]) => countB - countA,
        ),
      );
    });

    return res;
  }

  const genreNums = $derived(getGenreNums(movies, upYear));
  const genres = $derived(Object.keys(genreNums));

  type StackRow = {
    genre: string;
    [relatedGenre: string]: string | number;
  };

  const stackKeys = $derived(
    Array.from(
      new Set(
        Object.values(genreNums).flatMap((relatedGenres) =>
          Object.keys(relatedGenres),
        ),
      ),
    ),
  );

  const stackedData = $derived(
    Object.entries(genreNums).map(([genre, relatedGenres]) => {
      const row: StackRow = { genre };

      for (const key of stackKeys) {
        row[key] = relatedGenres[key] ?? 0;
      }

      return row;
    }),
  );

  const normalizedData = $derived(
    Object.entries(genreNums).map(([genre, relatedGenres]) => {
      const total = stackKeys.reduce(
        (sum, key) => sum + (relatedGenres[key] ?? 0),
        0,
      );

      const row: StackRow = { genre };

      for (const key of stackKeys) {
        const value = relatedGenres[key] ?? 0;

        // Avoid division by zero
        row[key] = total === 0 ? 0 : value / total;
      }

      return row;
    }),
  );
  const stackedSeries = $derived(
    d3
      .stack<StackRow>()
      .keys(stackKeys)
      .value((row, key) => Number(row[key] ?? 0))(normalizedData),
  );

  // drawing the bar chart

  const margin = {
    top: 15,
    bottom: 50,
    left: 30,
    right: 10,
  };

  let usableArea = {
    top: margin.top,
    right: width - margin.right,
    bottom: height - margin.bottom,
    left: margin.left,
  };

  const xScale = $derived(
    d3
      .scaleBand<string>()
      .range([usableArea.left, usableArea.right])
      .domain(normalizedData.map((d) => d.genre))
      .padding(0.17),
  );



  const yScale = $derived(
    d3
      .scaleLinear()
      .range([usableArea.bottom, usableArea.top])
      .domain([0, 1])
      .nice(),
  );

  const xBarwidth: number = $derived(xScale.bandwidth());

  let xAxis: any = $state(),
    yAxis: any = $state();

  function updateAxis() {
    d3.select(xAxis)
      .call(d3.axisBottom(xScale))
      .selectAll("text")
      .attr("transform", "rotate(45)")
      .style("text-anchor", "start");

    // tip:
    // similar to the x-axis, create a y-axis using d3.axisLeft() and bind it to the yAxis variable
    d3.select(yAxis).call(d3.axisLeft(yScale)).selectAll("text");
    //   .attr("transform", "rotate(45)")
    //   .style("text-anchor", "start");
  }

  // the $effect function is used to run a function whenever the reactive variables change, also known as a side effect
  $effect(() => {
    updateAxis();
  });
</script>

<h3>
  Correlation between Genres
</h3>

{#if movies.length > 0}
  <svg {width} {height}>
    <g class="bars">
      {#each stackedSeries as series}
        {#each series as segment, index}
          <rect
            class="bar"
            x={xScale(stackedData[index].genre)}
            y={yScale(segment[1])}
            width={xScale.bandwidth()}
            height={yScale(segment[0]) - yScale(segment[1])}
            fill={d3.schemeTableau10[series.index % 10]}
          />
        {/each}
      {/each}
    </g>

    <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
    <g transform="translate({usableArea.left}, 0)" bind:this={yAxis} />
  </svg>
  <div class="legend">
  {#each stackKeys as genre, index}
    <div class="legend-item">
      <span
        class="legend-color"
        style={`background-color: ${d3.schemeTableau10[index % 10]}`}
      ></span>
      <span>{genre}</span>
    </div>
  {/each}
</div>
{/if}

<style>
  .bar {
    transition:
      y 0.1s ease,
      height 0.1s ease,
      width 0.1s ease; /* Smooth transition for height */
  }
  .legend {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem 1rem;
  margin-top: 1rem;
  padding-left: 30px;
  max-width: 500px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 0.35rem;
  font-size: 0.85rem;
}

.legend-color {
  display: inline-block;
  width: 12px;
  height: 12px;
  border-radius: 2px;
  flex-shrink: 0;
}



</style>
