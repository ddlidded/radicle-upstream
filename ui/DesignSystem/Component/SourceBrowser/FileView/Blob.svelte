<script lang="typescript">
  import type { Blob } from "../../../../src/screen/project/source";

  import FileSource from "../FileSource.svelte";

  export let rootName: string;
  export let view: Blob;
</script>

<FileSource
  blob={view.blob}
  commit={view.blob.info.lastCommit}
  {rootName}
  on:root
  on:select />
