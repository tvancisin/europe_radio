<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import Hls from "hls.js";

  let audioElement;

  // Stream URLs for each country
  const streams = {
    uk: "https://as-hls-ww-live.akamaized.net/pool_904/live/ww/bbc_6music/bbc_6music.isml/bbc_6music-audio%3d96000.norewind.m3u8",
    austria: "https://orf-live.ors-shoutcast.at/fm4-q2a",
    portugal: "https://radiocast.rtp.pt/antena380a.mp3",
    slovakia: "https://icecast.stv.livebox.sk/fm_128.mp3",
    france: "https://icecast.radiofrance.fr/fip-midfi.mp3?id=radiofrance",
    spain: "https://rtvelivestream.akamaized.net/rtvesec/rne/rne_r3_main.m3u8",
    ireland: "https://icecast1.rte.ie/2xm",
    iceland: "http://netradio.ruv.is/ras2.aac",
    netherlands: "https://icecast.omroep.nl/3fm-bb-aac",
    czech: "https://rozhlas.stream/radio_wave_low.aac",
    switzerland: "https://stream.srg-ssr.ch/m/drsvirus/aacp_96",
    italy:
      "https://radiodueindie-live.akamaized.net/hls/live/2032593/radiodueindie/radiodueindie/radio2indie_256/chunklist.m3u8",
    germany: "https://st03.sslstream.dlf.de/dlf/03/high/aac/stream.aac",
    denmark:
      "https://drliveradio.akamaized.net/hls/live/2022411/p6beat/masterab.m3u8",
    poland: "http://mp3.polskieradio.pl:8904/;",
    hungary: "https://icast.connectmedia.hu/4738/mr2.mp3",
    slovenia: "https://mp3.rtvslo.si/val202",
    norway: "https://lyd.nrk.no/nrk_radio_p13_aac_h",
    sweden: "https://http-live.sr.se/p3-aac-192",
    finland: "https://icecast.live.yle.fi/radio/YleX/icecast.audio",
    estonia: "https://sb.err.ee/live/raadio2.m3u8",
    latvia: "https://live.pieci.lv/live19-hq.aac",
    lithuania: "http://lrt-cast.lrt.lt:8000/lrt_opus",
    croatia:
      "https://playerservices.streamtheworld.com/api/livestream-redirect/PROGRAM2AAC.aac",
    bosnia: "https://pstnet7.shoutcastnet.com:10034/stream",
  };

  let width, height;
  let transform = d3.zoomIdentity;
  let imageSize = 25; // Default size for images
  let svgElement;
  let zoomBehavior;
  const defaultCountryFill = "#1B1A1B";
  const selectedCountryFill = "#3A383A";
  const hoverCountryFill = "red";
  let selectedCountryName;
  let hoveredCountryName;

  $: imageSize = width < 600 ? 15 : 25; // Adjust size based on screen width

  let geojson;
  d3.json("europe.json").then((data) => (geojson = data));

  $: projection = d3
    .geoOrthographic()
    .rotate([-15, -54])
    .translate([width / 2, height / 2])
    .scale(Math.min(width, height) * 1.5)
    .clipAngle(90)
    .precision(0.3);


  $: pathGenerator = d3.geoPath(projection);

  let countries = [];
  $: if (geojson)
    countries = geojson.features.map((feature) => {
      return {
        ...feature,
        path: pathGenerator(feature),
        bounds: pathGenerator.bounds(feature),
        centroid: pathGenerator.centroid(feature), // Get the country's central point
      };
    });

  function smoothZoom(newTransform) {
    if (!svgElement || !zoomBehavior) {
      transform = newTransform;
      return;
    }

    d3.select(svgElement)
      .transition()
      .duration(750)
      .call(zoomBehavior.transform, newTransform);
  }

  function handleClick(e) {
    const countryName = e.properties.NAME;
    selectedCountryName = countryName;

    let streamCountryKey = countryName;
    if (streamCountryKey == "United Kingdom") {
      streamCountryKey = "uk";
    }
    if (streamCountryKey == "Czech Republic") {
      streamCountryKey = "czech";
    }
    if (streamCountryKey == "Bosnia and Herzegovina") {
      streamCountryKey = "bosnia";
    }

    const streamUrl = streams[streamCountryKey.toLowerCase()];
    if (streamUrl) {
      playStream(streamUrl);
    } else {
      console.error(`No stream available for ${streamCountryKey}`);
    }

    const [[x0, y0], [x1, y1]] = e.bounds;
    const newTransform = d3.zoomIdentity
      .translate(width / 2, height / 2)
      .scale(Math.min(3, 0.9 / Math.max((x1 - x0) / width, (y1 - y0) / height)))
      .translate(-(x0 + x1) / 2, -(y0 + y1) / 2);

    smoothZoom(newTransform);
  }

  function playStream(url) {
    // Check if the stream is HLS or MP3
    if (url.endsWith(".m3u8")) {
      // Handle HLS streams
      if (Hls.isSupported()) {
        const hls = new Hls();
        hls.loadSource(url);
        hls.attachMedia(audioElement);
        hls.on(Hls.Events.MANIFEST_PARSED, () => {
          console.log("HLS Manifest loaded, ready to play.");
          audioElement.play();
        });
        hls.on(Hls.Events.ERROR, (event, data) => {
          console.error("HLS Error:", data);
        });
      } else if (audioElement.canPlayType("application/vnd.apple.mpegurl")) {
        // Safari handles HLS natively
        audioElement.src = url;
        audioElement.play();
      } else {
        console.error("Your browser does not support HLS playback.");
      }
    } else {
      // Handle MP3 streams
      audioElement.src = url;
      audioElement.play();
    }
  }

  function handleEscape(event) {
    if (event.key === "Escape") {
      smoothZoom(d3.zoomIdentity);
    }
  }

  $: if (svgElement && width && height) {
    zoomBehavior = d3
      .zoom()
      .scaleExtent([1, 8])
      .translateExtent([
        [0, 0],
        [width, height],
      ])
      .on("zoom", (event) => {
        transform = event.transform;
      });

    d3.select(svgElement).call(zoomBehavior);
  }

  onMount(() => {
    window.addEventListener("keydown", handleEscape);
    return () => window.removeEventListener("keydown", handleEscape);
  });

  function handleMouseEnter(country) {
    hoveredCountryName = country.properties.NAME;
  }

  function handleMouseLeave(country) {
    if (hoveredCountryName === country.properties.NAME) {
      hoveredCountryName = undefined;
    }
  }
</script>

{#if countries}
  <main bind:clientWidth={width} bind:clientHeight={height}>
    <audio bind:this={audioElement} controls autoplay></audio>

    <svg bind:this={svgElement} {width} {height}>
      <g transform={transform}>
        {#each countries as country}
          <path
            fill={hoveredCountryName === country.properties.NAME
              ? hoverCountryFill
              : selectedCountryName === country.properties.NAME
                ? selectedCountryFill
                : defaultCountryFill}
            stroke="white"
            stroke-width="0.05"
            d={country.path}
            role="button"
            aria-label={`Select ${country.properties.NAME}`}
            tabindex="0"
            on:click={() => handleClick(country)}
            on:mouseenter={() => handleMouseEnter(country)}
            on:mouseleave={() => handleMouseLeave(country)}
            on:keydown={(event) => {
              if (event.key === "Enter" || event.key === " ") {
                handleClick(country);
                event.preventDefault(); // Prevent default scrolling for space
              }
            }}
          ></path>
          <!-- <image
            on:click={() => handleClick(country)}
            on:keydown={(event) => {
              if (event.key === "Enter" || event.key === " ") {
                handleClick(country);
                event.preventDefault(); // Prevent scrolling with Space
              }
            }}
            href={country.properties.radio}
            x={country.centroid[0] - imageSize / 2}
            y={country.centroid[1] - imageSize / 2}
            height={imageSize}
            role="button"
            tabindex="0"
            aria-label={`Play radio for ${country.properties.NAME}`}
          /> -->
        {/each}
      </g>
    </svg>
  </main>
{/if}

<style>
  main {
    width: 100vw;
    height: 100vh;
    cursor: pointer;
    overflow: hidden;
  }
  audio {
    position: absolute;
    right: 5px;
    top: 5px;
  }
  /* Remove focus outline for mouse users */
  path:focus {
    outline: none;
  }
</style>
