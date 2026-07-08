# CHESS-TOURNAMENT-MANAGEMENT-SYSTEM-
<script>
  // --- IN-MEMORY DATABASE SCHEMAS & STATE ---
  let players = [
    { id: 1, name: "Magnus Carlsen", elo: 2882, wins: 15, losses: 2 },
    { id: 2, name: "Hikaru Nakamura", elo: 2875, wins: 12, losses: 4 },
    { id: 3, name: "Fabiano Caruana", elo: 2804, wins: 10, losses: 5 },
    { id: 4, name: "Ding Liren", elo: 2780, wins: 8, losses: 7 },
    { id: 5, name: "Praggnanandhaa R", elo: 2750, wins: 9, losses: 4 }
  ];

  let tournaments = [
    { id: 1, name: "Grand Chess Tour 2026", location: "Zagreb", status: "Active", playerIds: [1, 2, 3] },
    { id: 2, name: "FIDE Candidates", location: "Toronto", status: "Upcoming", playerIds: [1, 4] }
  ];

  let matches = [];

  // --- FORM STATES ---
  // Player forms
  let playerForm = { id: null, name: "", elo: 1500 };
  let isEditingPlayer = false;

  // Tournament forms
  let tournamentForm = { id: null, name: "", location: "", status: "Upcoming" };
  let isEditingTournament = false;
  let selectedTournamentId = 1;
  let playerToAddToTournament = "";

  // --- PLAYER CRUD OPERATIONS ---
  function savePlayer() {
    if (!playerForm.name.trim()) return;
    
    if (isEditingPlayer) {
      players = players.map(p => p.id === playerForm.id ? { ...p, name: playerForm.name, elo: Number(playerForm.elo) } : p);
      isEditingPlayer = false;
    } else {
      const newId = players.length > 0 ? Math.max(...players.map(p => p.id)) + 1 : 1;
      players = [...players, { id: newId, name: playerForm.name, elo: Number(playerForm.elo), wins: 0, losses: 0 }];
    }
    resetPlayerForm();
  }

  function editPlayer(player) {
    playerForm = { ...player };
    isEditingPlayer = true;
  }

  function deletePlayer(id) {
    players = players.filter(p => p.id !== id);
    // Cascade delete player from tournament rosters
    tournaments = tournaments.map(t => ({
      ...t,
      playerIds: t.playerIds.filter(pid => pid !== id)
    }));
  }

  function resetPlayerForm() {
    playerForm = { id: null, name: "", elo: 1500 };
    isEditingPlayer = false;
  }

  // --- TOURNAMENT CRUD OPERATIONS ---
  function saveTournament() {
    if (!tournamentForm.name.trim() || !tournamentForm.location.trim()) return;

    if (isEditingTournament) {
      tournaments = tournaments.map(t => t.id === tournamentForm.id ? { ...t, name: tournamentForm.name, location: tournamentForm.location, status: tournamentForm.status } : t);
      isEditingTournament = false;
    } else {
      const newId = tournaments.length > 0 ? Math.max(...tournaments.map(t => t.id)) + 1 : 1;
      tournaments = [...tournaments, { id: newId, name: tournamentForm.name, location: tournamentForm.location, status: tournamentForm.status, playerIds: [] }];
    }
    resetTournamentForm();
  }

  function editTournament(tournament) {
    tournamentForm = { ...tournament };
    isEditingTournament = true;
  }

  function deleteTournament(id) {
    tournaments = tournaments.filter(t => t.id !== id);
    matches = matches.filter(m => m.tournamentId !== id);
  }

  function resetTournamentForm() {
    tournamentForm = { id: null, name: "", location: "", status: "Upcoming" };
    isEditingTournament = false;
  }

  // --- RELATIONSHIP MANAGEMENT (Add Player to Tournament) ---
  function addPlayerToTournament() {
    if (!playerToAddToTournament) return;
    const playerId = Number(playerToAddToTournament);
    
    tournaments = tournaments.map(t => {
      if (t.id === selectedTournamentId) {
        if (!t.playerIds.includes(playerId)) {
          return { ...t, playerIds: [...t.playerIds, playerId] };
        }
      }
      return t;
    });
    playerToAddToTournament = "";
  }

  // --- RANDOM MATCH SYSTEM ENGINE ---
  function generateRandomMatch(tournamentId) {
    const tournament = tournaments.find(t => t.id === tournamentId);
    if (!tournament || tournament.playerIds.length < 2) {
      alert("At least 2 players must be registered in this tournament to simulate a matchup!");
      return;
    }

    // Pick two random distinct players from the tournament roster
    let shuffled = [...tournament.playerIds].sort(() => 0.5 - Math.random());
    const player1Id = shuffled[0];
    const player2Id = shuffled[1];

    const p1 = players.find(p => p.id === player1Id);
    const p2 = players.find(p => p.id === player2Id);

    // Randomly select a winner (0 = Draw, 1 = Player 1 Wins, 2 = Player 2 Wins)
    const rng = Math.random();
    let winnerId = null;
    let resultText = "Draw";

    if (rng < 0.45) {
      winnerId = player1Id;
      resultText = `${p1.name} Wins`;
      players = players.map(p => p.id === player1Id ? { ...p, wins: p.wins + 1 } : p.id === player2Id ? { ...p, losses: p.losses + 1 } : p);
    } else if (rng < 0.90) {
      winnerId = player2Id;
      resultText = `${p2.name} Wins`;
      players = players.map(p => p.id === player2Id ? { ...p, wins: p.wins + 1 } : p.id === player1Id ? { ...p, losses: p.losses + 1 } : p);
    } else {
      resultText = "Match Drawn";
    }

    const newMatch = {
      id: matches.length + 1,
      tournamentId,
      tournamentName: tournament.name,
      player1Name: p1.name,
      player2Name: p2.name,
      result: resultText,
      timestamp: new Date().toLocaleTimeString()
    };

    matches = [newMatch, ...matches];
  }

  // --- DYNAMIC RANKING CALCULATIONS ---
  $: rankedPlayers = [...players].sort((a, b) => b.wins - a.wins || b.elo - a.elo);
  $: currentTournament = tournaments.find(t => t.id === selectedTournamentId);
</script>

<main class="app-container">
  <header class="hero-banner">
    <h1>🏆 Grandmaster Tournament Matrix</h1>
    <p>Real-Time Chess Core Matchmaking & Roster Subsystem</p>
  </header>

  <div class="grid-layout">
    
    <!-- LEFT PANEL: ROSTER AND PLAYER CRUD -->
    <section class="card entries-card">
      <h2>👥 Player Roster Management</h2>
      
      <form on:submit|preventDefault={savePlayer} class="form-container">
        <input type="text" placeholder="Full Player Name" bind:value={playerForm.name} required />
        <input type="number" placeholder="ELO Rating" bind:value={playerForm.elo} required min="100" max="3500" />
        <div class="button-group">
          <button type="submit" class="btn primary">{isEditingPlayer ? "✏️ Update Player" : "➕ Add Player"}</button>
          {#if isEditingPlayer}
            <button type="button" class="btn secondary" on:click={resetPlayerForm}>Cancel</button>
          {/if}
        </div>
      </form>

      <div class="table-scroll">
        <table>
          <thead>
            <tr>
              <th>ID</th>
              <th>Name</th>
              <th>ELO</th>
              <th>Record (W-L)</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {#each players as player}
              <tr>
                <td>#{player.id}</td>
                <td class="bold-text">{player.name}</td>
                <td><span class="badge elo">{player.elo}</span></td>
                <td><span class="badge win">{player.wins}W</span> - <span class="badge loss">{player.losses}L</span></td>
                <td>
                  <button class="icon-btn edit" on:click={() => editPlayer(player)}>✏️</button>
                  <button class="icon-btn delete" on:click={() => deletePlayer(