<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import { Bar } from "$lib";
  import type { TMovie } from "../../types";

  // Reactive variable for storing the data
  let movies: TMovie[] = [];

  // Function to load the CSV
  async function loadCsv() {
    try {
      const csvUrl = "./summer_movies.csv";
      movies = await d3.csv(csvUrl, (row) => {
        // TIP: in row, all values are strings, so we need to use a row conversion function here to format them
        return {
          // ...row, // spread syntax to copy all properties from row
          // num_votes: Number(row.num_votes),
          // year: new Date(row.year),
          // please also format the values for other non-string attributes. You can check the attributes in the CSV file
          ...row,
          num_votes: Number(row.num_votes),
          runtime_minutes:  Number(row.runtime_minutes),
          genres: row.genres.split(",").map((genre) => genre.trim()),
          year: new Date(row.year),
          average_rating: Number(row.average_rating),
        };
      });

      console.log("Loaded CSV Data:", movies);
    } catch (error) {
      console.error("Error loading CSV:", error);
    }
  }
  // Call the loader when the component mounts
  onMount(loadCsv);
</script>

<h1>Summer Movies</h1>

<p>Here are {movies.length == 0 ? "..." : movies.length + " "} movies</p>
 <Bar {movies} />
