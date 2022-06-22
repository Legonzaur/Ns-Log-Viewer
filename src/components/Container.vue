<template>
  <input type="file" @change="loadFile" />
  <div v-for="match in text" v-bind:key="match">
    <div v-for="message in match" v-bind:key="message">{{ message }}</div>
  </div>
</template>

<script lang="ts">
import { Options, Vue } from "vue-class-component";

const regexes = {
  matchStart:
    /^\[([0-9]{2}:){2}[0-9]{2}\] \[info] \[SERVER SCRIPT\] Code Script: _init$/gs,
  joinRequest:
    /^(?:\[(?:[0-9]{2}:){2}[0-9]{2}\] \[info] Player ")([a-zA-Z]*)(?:" with UID "([0-9]*)" requested to join)$/gs,
  join: /\[(?:[0-9]{2}:){2}[0-9]{2}\] \[info] \[SERVER SCRIPT] Player client script initialization complete: entity \([0-9]: player ([a-zA-Z]*) \[[0-9]]\)/gm,
  message:
    /\[(?:[0-9]{2}:){2}[0-9]{2}\] \[info] \[SERVER SCRIPT] Received message from entity \([0-9]: player ([a-zA-Z]*) \[[0-9]]\)\(([0-9]*)\): (.*)/gm,
};

//Stub for worker or some other async stuff
function* readFileByLine(file: string) {
  const iterator = file.split("\r\n");
  const length = iterator.length;
  for (let i = 0; i < length; i++) {
    yield iterator[i];
  }
}

function parseLines(file: string) {
  const fileHandler = readFileByLine(file);
  let result;
  let regexOutput;
  let matches: Array<Array<string>> = [];
  do {
    result = fileHandler.next();
    if (!result.value) continue;
    if (regexes.matchStart.test(result.value)) {
      matches.push([]);
      matches[matches.length - 1].push("New Match started");
      continue;
    }
    regexOutput = regexes.joinRequest.exec(result.value);
    if (regexOutput) {
      matches[matches.length - 1].push(
        `${regexOutput?.[1]}#${regexOutput?.[2]} requested to join the game`
      );
      continue;
    }
    regexOutput = regexes.join.exec(result.value);
    if (regexOutput) {
      matches[matches.length - 1].push(`${regexOutput?.[1]} joined the game`);
      continue;
    }
    regexOutput = regexes.message.exec(result.value);
    if (regexOutput) {
      matches[matches.length - 1].push(
        `${regexOutput?.[1]}#${regexOutput?.[2]} : ${regexOutput?.[3]}`
      );
      continue;
    }
    /*if (matches.length < 1) continue;
        matches[matches.length - 1].push(result.value);*/
  } while (!result.done);
  return matches;
}

@Options({
  computed: {
    text() {
      return parseLines(this.log);
    },
  },
})
export default class Container extends Vue {
  log!: string;
  data() {
    return { log: "" };
  }

  loadFile(e: Event) {
    const file = (e.target as HTMLInputElement)?.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.readAsText(file);
    reader.addEventListener(
      "load",
      () => {
        this.log = reader.result as string;
      },
      false
    );
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
