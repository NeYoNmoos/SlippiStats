<script setup>
import { ref, onMounted, watch } from "vue";

const gamesData = ref(null);
const results = ref({});

onMounted(() => {
  console.log("Start app!");

  fetch("../../data/replayCache.json")
    .then((response) => response.json())
    .then((json) => {
      gamesData.value = json;
    });
});

watch(gamesData, (newData) => {
  if (newData) {
    processData(newData);
  }
});

function processData(data) {
  const cache = { results: {} };
  let total_games = 0;
  let total_wins = 0;
  let total_seconds = 0;
  let counted_seconds = 0;
  let num = 0;
  // ... Other required data structures and constants
  console.log("logging data");
  console.log(data);

  Object.entries(data.results).forEach(([gameId, gameData], i) => {
    // const gameData = loadGameData(file, i);
    if (!gameData) {
      return;
    }
    let file = "";
    // cache.results[gameData.hash] = gameData;
    const gameResults = processGame(file, i, gameData);
    processResults(gameResults);
  });

  function processGame(file, i, gameData) {
    const { settings, metadata, stats, latestFramePercents } = gameData;
    let data = {};
    num = i + 1;
    console.log("process game");
    // console.log(settings, metadata, stats, latestFramePercents);
    console.log(metadata);
    console.log(metadata.lastFrame);

    let game_seconds = 0;
    let game_length = 0;

    try {
      try {
        game_seconds = Math.floor(metadata.lastFrame / 60);
        game_length =
          Math.floor(game_seconds / 60) +
          ":" +
          (game_seconds % 60
            ? (game_seconds % 60).toString().padStart(2, "0")
            : "00");
        data.total_seconds = game_seconds;
      } catch (err) {
        console.log(err);
        console.log(`${num}: Error reading metadata. Ignoring... (${file})`);
        return data;
      }
      if (settings.players.length !== 2) {
        console.log(`${num}: More than 2 players. Ignoring... (${file})`);
        return data;
      }
      try {
        if (
          JSON.stringify(metadata.players[0].names) === "{}" ||
          JSON.stringify(metadata.players[1].names) === "{}"
        ) {
          console.log(`${num}: Old or offline. Ignoring... (${file})`);
          return data;
        }
      } catch (err) {
        console.log(`${num}: Missing player info. Ignoring... (${file})`);
        return data;
      }

      let player_num = "none";
      let opponent_num = "none";
      let opponent_found = false;
      let ignored_opponent_found = false;
      let player_names = [
        metadata.players[0].names.netplay,
        metadata.players[1].names.netplay,
      ];
      let player_codes = [
        metadata.players[0].names.code,
        metadata.players[1].names.code,
      ];
      let player_characters = [
        settings.players[0].characterId,
        settings.players[1].characterId,
      ];

      let opponent_arg = "";
      let ignored_arg = "";
      let user_player = "";

      for (let j = 0; j < settings.players.length; j++) {
        if (opponent_arg) {
          for (k of opponent_player) {
            opponent = k.trim();
            if (
              player_names[j].toLowerCase().trim() == opponent ||
              player_codes[j].toLowerCase() == opponent
            ) {
              opponent_found = true;
            }
          }
        }
        if (ignored_arg) {
          for (k of ignored_list) {
            skipped_opponent = k.trim();
            if (
              player_names[j].toLowerCase().trim() == skipped_opponent ||
              player_codes[j].toLowerCase() == skipped_opponent
            ) {
              ignored_opponent_found = true;
              found_ignored_opponent = `${player_names[j]} (${player_codes[j]})`;
            }
          }
        }
        for (k of user_player) {
          if (
            player_names[j].toLowerCase().trim() == k ||
            player_codes[j].toLowerCase() == k
          ) {
            player_num = j;
            if (player_num == 0) {
              opponent_num = 1;
            } else {
              opponent_num = 0;
            }
            break;
          }
        }
      }
      if (player_num == "none") {
        console.log(
          `${num}: User(s) ${user_player} missing. Ignoring... (${file})`
        );
        return data;
      }
      if (opponent_arg && !opponent_found) {
        console.log(
          `${num}: Opponent(s) ${opponent_player} missing. Ignoring... (${file})`
        );
        return data;
      }
      if (ignored_arg && ignored_opponent_found) {
        console.log(
          `${num}: ${found_ignored_opponent} found. Ignoring... (${file})`
        );
        return data;
      }

      player_character_num = player_characters[player_num];
      player_character = characters[player_character_num];
      player_name = player_names[player_num];

      opponent_character_num = player_characters[opponent_num];
      opponent_character = characters[opponent_character_num];
      opponent_name = player_names[opponent_num];
      opponent_code = player_codes[opponent_num];

      stage_num = settings.stageId;

      if (
        player_character_arg &&
        player_character.toLowerCase() !== player_character_requested
      ) {
        requested_player_character_num = characters_lowercase.indexOf(
          player_character_requested
        );
        console.log(
          `${num}: ${player_name} playing ${player_character}. Ignoring... (${file})`
        );
        return data;
      }

      if (
        character_arg &&
        opponent_character.toLowerCase() !== character_requested
      ) {
        requested_character_num =
          characters_lowercase.indexOf(character_requested);
        console.log(
          `${num}: ${opponent_name} playing ${opponent_character}. Ignoring... (${file})`
        );
        return data;
      }

      player_kills = stats[player_num];
      opponent_kills = stats[opponent_num];

      // Tie conditions
      if (game_seconds < 30) {
        console.log(`${num}: Match under 30 seconds. Ignoring... (${file})`);
        return data;
      }
      if (player_kills == 0 && opponent_kills == 0) {
        console.log(`${num}: No stocks were taken. Ignoring... (${file})`);
        return data;
      }

      player_final_percent = latestFramePercents[player_num];
      opponent_final_percent = latestFramePercents[opponent_num];
      end_more_kills = player_kills > opponent_kills;
      end_lower_percent =
        player_kills == opponent_kills &&
        player_final_percent < opponent_final_percent;

      // disabled due to perceived inconsistencies
      // try {
      //     end_opponent_LRAS = (game.getGameEnd().lrasInitiatorIndex == opponent_num)
      //     end_player_LRAS = (game.getGameEnd().lrasInitiatorIndex == player_num)
      // }
      // catch {
      //     end_opponent_LRAS = false
      //     end_player_LRAS = false
      // }

      // Every death is considered the opponent's kill
      // If the player didn't quit out AND has more kills than the opponent, the same but with a lower percent, or the opponent quits out: it's a win, otherwise it's a loss. Ties handled above
      // if (!end_player_LRAS && (end_more_kills || end_lower_percent || end_opponent_LRAS)) {
      if (end_more_kills || end_lower_percent) {
        console.log(
          `${num}: ${
            player_name || player_codes[player_num]
          } (${player_character}) \x1b[36mwon\x1b[0m vs ${
            opponent_name || opponent_code
          } (${opponent_character}) on ${
            stages[stage_num]
          } in ${game_length}! (${file})`
        );
        data.total_wins = 1;
      } else {
        console.log(
          `${num}: ${
            player_name || player_codes[player_num]
          } (${player_character}) \x1b[31mlost\x1b[0m vs ${
            opponent_name || opponent_code
          } (${opponent_character}) on ${
            stages[stage_num]
          } in ${game_length}. (${file})`
        );
      }

      data.total_games = 1;
      data.player_character_num = player_character_num;
      data.player_name = player_name;
      data.opponent_character_num = opponent_character_num;
      data.opponent_code = opponent_code;
      data.stage_num = settings.stageId;

      if (player_name.length > 0) {
        data.player_name = player_name;
      }
      data.player_code = player_codes[player_num];
      if (opponent_arg && player_names[opponent_num]) {
        if (opponent_name.length > 0) {
          data.opponent_name = opponent_name;
        }
        data.opponent_code = player_codes[opponent_num];
      }
      data.game_seconds = game_seconds;
      return data;
    } catch (err) {
      console.log(err);
      console.log(`${num}: Error reading replay. Ignoring... (${file})`);
      return data;
    }
  }

  function processResults(r) {
    total_games += r.total_games || 0;
    total_wins += r.total_wins || 0;
    total_seconds += r.total_seconds || 0;
    counted_seconds += r.game_seconds || 0;

    // Try to find last used nickname and actual connect code to display at the end
    if (!!r.player_code) {
      real_player_code = r.player_code;
    }
    if (!!r.player_name) {
      final_player_name = r.player_name;
    }
    if (!!r.opponent_name) {
      final_opponent_name = r.opponent_name;
    }
    if (!!r.opponent_code) {
      real_opponent_code = r.opponent_code;
    }

    if (!!r.player_character_num || r.player_character_num == 0) {
      character_totals[r.player_character_num] =
        character_totals[r.player_character_num] + 1 || 1;
      character_playtime[r.player_character_num] =
        character_playtime[r.player_character_num] + r.game_seconds ||
        r.game_seconds;
    }

    if (!!r.player_name) {
      nickname_totals[r.player_name] = nickname_totals[r.player_name] + 1 || 1;
      nickname_playtime[r.player_name] =
        nickname_playtime[r.player_name] + r.game_seconds || r.game_seconds;
    }

    if (!!r.player_code) {
      code_totals[r.player_code] = code_totals[r.player_code] + 1 || 1;
      code_playtime[r.player_code] =
        code_playtime[r.player_code] + r.game_seconds || r.game_seconds;
    }

    if (!!r.opponent_code) {
      opponent_totals[r.opponent_code] =
        opponent_totals[r.opponent_code] + 1 || 1;
      opponent_playtime[r.opponent_code] =
        opponent_playtime[r.opponent_code] + r.game_seconds || r.game_seconds;
    }

    if (!!r.stage_num) {
      stage_totals[r.stage_num] = stage_totals[r.stage_num] + 1 || 1;
      stage_playtime[r.stage_num] =
        stage_playtime[r.stage_num] + r.game_seconds || r.game_seconds;
    }

    // If the player won the game
    if (r.total_wins == 1) {
      character_wins[r.player_character_num] =
        character_wins[r.player_character_num] + 1 || 1;
      character_head_to_head[r.player_character_num][
        r.opponent_character_num
      ][2] = characters[r.opponent_character_num];
      character_head_to_head[r.player_character_num][
        r.opponent_character_num
      ][0] =
        character_head_to_head[r.player_character_num][
          r.opponent_character_num
        ][0] + 1;
      nickname_wins[r.player_name] = nickname_wins[r.player_name] + 1 || 1;
      code_wins[r.player_code] = code_wins[r.player_code] + 1 || 1;
      opponent_wins[r.opponent_code] = opponent_wins[r.opponent_code] + 1 || 1;
      stage_wins[r.stage_num] = stage_wins[r.stage_num] + 1 || 1;
    }
    // Else if the player lost the game and it wasn't filtered
    else if (!!r.total_games) {
      character_head_to_head[r.player_character_num][
        r.opponent_character_num
      ][2] = characters[r.opponent_character_num];
      character_head_to_head[r.player_character_num][
        r.opponent_character_num
      ][1] =
        character_head_to_head[r.player_character_num][
          r.opponent_character_num
        ][1] + 1;
    }
  }

  // store results in the results ref
  results.value = {
    total_games,
    total_wins,
    total_seconds,
    counted_seconds,
    // ... other required results
  };
}
</script>

<template>
  <div>
    <!-- Display your results here -->
    <pre>{{ results }}</pre>
  </div>
</template>

<style scoped></style>
