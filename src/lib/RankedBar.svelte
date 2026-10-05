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

type GenreCounts = Record<string, number>;
type GenreNumsByYear = Record<number, GenreCounts>;

type RankedGenre = {
  genre: string;
  rank: number;
};

type RankedGenresByYear = Record<number, Record<string, number>>;

function getGenreNums(
  movies: TMovie[],
  upYear: Date
): RankedGenresByYear {
  const counts: Record<number, Record<string, number>> = {};

  movies
    .filter((movie) => movie.year <= upYear)
    .forEach((movie) => {
      const year = movie.year.getFullYear();

      counts[year] ??= {};

      movie.genres.forEach((genre: string) => {
        counts[year][genre] = (counts[year][genre] || 0) + 1;
      });
    });

  const rankedGenres: RankedGenresByYear = {};

  Object.entries(counts).forEach(([year, genres]) => {
    rankedGenres[Number(year)] = Object.fromEntries(
      Object.entries(genres)
        .sort(([, countA], [, countB]) => countB - countA)
        .slice(0, 3)
        .map(([genre], index) => [genre, index + 1])
    );
  });

  return rankedGenres;
}




  const genreNums = $derived(getGenreNums(movies, upYear));

  type StackRow = {
    genre: string;
    [relatedGenre: string]: string | number;
  };

const stackKeys = $derived(
  Array.from(
    new Set(
      Object.values(genreNums).flatMap((genresForYear) =>
        Object.keys(genresForYear),
      ),
    ),
  ).sort((genreA, genreB) => {
    // Find each genre's best rank across all years
    const rankA = d3.min(
      Object.values(genreNums),
      (genresForYear) => genresForYear[genreA] ?? Infinity,
    ) ?? Infinity;

    const rankB = d3.min(
      Object.values(genreNums),
      (genresForYear) => genresForYear[genreB] ?? Infinity,
    ) ?? Infinity;

    return rankA - rankB;
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

  type StackSegment = {
  genre: string;
  rank: number;
  y0: number;
  y1: number;
};

type StackedBar = {
  year: string;
  segments: StackSegment[];
};

const allGenres = $derived(
  Array.from(
    new Set(
      Object.values(genreNums).flatMap((genresForYear) =>
        Object.keys(genresForYear),
      ),
    ),
  ),
);

const genreColor = $derived(
  d3
    .scaleOrdinal<string, string>()
    .domain(allGenres)
    .range(d3.schemeTableau10),
);


const stackedBars = $derived<StackedBar[]>(
  Object.entries(genreNums).map(([year, genresForYear]) => {
    // Rank 1 should be at the top.
    // Therefore, draw the lowest-ranked genre first.
    const sortedGenres = Object.entries(genresForYear).sort(
      ([, rankA], [, rankB]) => rankB - rankA,
    );

    const total = sortedGenres.length;
    let cumulative = 0;

    const segments = sortedGenres.map(([genre, rank]) => {
      const y0 = cumulative;
      const y1 = cumulative + 1 / total;

      cumulative = y1;

      return {
        genre,
        rank,
        y0,
        y1,
      };
    });

    return {
      year,
      segments,
    };
  }),
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
      .padding(0),
  );



const yScale = $derived(
  d3
    .scaleLinear()
    .domain([0, 1])
    .range([usableArea.bottom, usableArea.top]),
);


  const xBarwidth: number = $derived(xScale.bandwidth());

  let xAxis: any = $state(),
    yAxis: any = $state();

function updateAxis() {
  const years = xScale.domain();

  const tickYears = years.filter(
    (year) => Number(year) % 5 === 0,
  );

  d3.select(xAxis)
    .call(
      d3
        .axisBottom(xScale)
        .tickValues(tickYears),
    )
    .selectAll("text")
    .attr("transform", "rotate(45)")
    .style("text-anchor", "start");

  d3.select(yAxis).call(
    d3
      .axisLeft(yScale)
      .tickValues([0, 1 / 3, 2 / 3, 1])
      .tickFormat((value) => {
        if (value === 1) return "1st";
        if (value === 2 / 3) return "2nd";
        if (value === 1 / 3) return "3rd";
        return "";
      }),
  );
}



  // the $effect function is used to run a function whenever the reactive variables change, also known as a side effect
  $effect(() => {
    updateAxis();
  });
</script>

<h3>
Top 3 Genres by Year
</h3>

{#if movies.length > 0}

  <svg {width} {height} >
  <g class="bars">
  {#each stackedBars as bar}
    {#each bar.segments as segment, index}
      <rect
        class="bar"
        x={xScale(bar.year)}
        y={yScale(segment.y1)}
        width={xScale.bandwidth()}
        height={yScale(segment.y0) - yScale(segment.y1)}
        fill={genreColor(segment.genre)}
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
