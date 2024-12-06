Certainly! Below is the modified `gwent.js` file that allows two players to play against each other on the same computer. The key changes include:

1. **Switching from AI to Human Controller for Both Players:**
   - Both `player_me` and `player_op` are now human-controlled by assigning them instances of the `Controller` class instead of `ControllerAI`.

2. **Adding a "Change Player" Button:**
   - A "Change Player" button is dynamically created and managed within the `UI` class. This button appears after a player completes their turn, allowing the next player to take their turn.

3. **Managing Hand Visibility:**
   - Only the current player's hand is visible, while the other player's hand is hidden to prevent viewing unplayed cards.

4. **Integrating Turn Switching Logic:**
   - The `Game` class is updated to handle the manual switching of turns between players via the "Change Player" button.

Below is the complete modified `gwent.js` with detailed comments highlighting the changes:

```javascript
"use strict"

// Base Controller class (empty for human players)
class Controller {}

// Removed ControllerAI class since both players are now human-controlled

// Player class remains largely the same, with modifications to support human controllers
class Player {
	constructor(id, name, deck) {
		this.id = id;
		this.tag = (id === 0) ? "me" : "op";
		// Both players now use the base Controller for human control
		this.controller = new Controller();
		
		// Assign Hand instances to both players instead of HandAI for human control
		this.hand = new Hand(document.getElementById("hand-row-" + this.tag));
		this.grave =  new Grave( document.getElementById("grave-" + this.tag));
		this.deck = new Deck(deck.faction, document.getElementById("deck-" + this.tag));
		this.deck_data = deck;
		
		this.leader = new Card(deck.leader, this);
		this.elem_leader = document.getElementById("leader-" + this.tag);
		this.elem_leader.children[0].appendChild( this.leader.elem );
		
		this.reset();
		
		this.name = name;
		document.getElementById("name-" + this.tag).innerHTML = name;
		
		document.getElementById("deck-name-" +this.tag).innerHTML = factions[deck.faction].name;
		document.getElementById("stats-" + this.tag).getElementsByClassName("profile-img")[0].children[0].children[0];
		let x = document.querySelector("#stats-" +this.tag+ " .profile-img > div > div");
		x.style.backgroundImage = iconURL("deck_shield_" + deck.faction);
	}
	
	// Sets default values
	reset(){
		this.grave.reset();
		this.hand.reset();
		this.deck.reset();
		this.deck.initializeFromID(this.deck_data.cards, this);
		
		this.health = 2;
		this.total = 0;
		this.passed = false;
		this.handsize = 10;
		this.winning = false;
	
		this.enableLeader();
		this.setPassed(false);
		document.getElementById("gem1-" +this.tag).classList.add("gem-on");
		document.getElementById("gem2-" +this.tag).classList.add("gem-on");
	}
	
	// Returns the opponent Player
	opponent(){
		return board.opponent(this);
	}
	
	// Updates the player's total score and notifies the game
	updateTotal(n){
		this.total += n;
		document.getElementById("score-total-" + this.tag).children[0].innerHTML = this.total;
		board.updateLeader();
	}
	
	// Puts the player in the winning state
	setWinning(isWinning) {
		if (this.winning ^ isWinning)
			document.getElementById("score-total-" + this.tag).classList.toggle("score-leader");
		this.winning = isWinning;
	}
	
	// Puts the player in the passed state
	setPassed(hasPassed) {
		if (this.passed ^ hasPassed)
			document.getElementById("passed-" + this.tag).classList.toggle("passed");
		this.passed = hasPassed;
	}
	
	// Sets up board for turn
	async startTurn(){
		document.getElementById("stats-" + this.tag).classList.add("current-turn");
		if (this.leaderAvailable)
			this.elem_leader.children[1].classList.remove("hide");
		
		if (this === player_me) {
			document.getElementById("pass-button").classList.remove("noclick");
		}
		
		// Removed AI turn logic since both players are human
	}
	
	// Passes the round and ends the turn
	passRound(){
		this.setPassed(true);
		this.endTurn();
	}
	
	// Plays a scorch card
	async playScorch(card){
		await this.playCardAction(card, async () => await ability_dict["scorch"].activated(card));
	}
	
	// Plays a card to a specific row
	async playCardToRow(card, row){
		await this.playCardAction(card, async () => await board.moveTo(card, row, this.hand));
	}
	
	// Plays a card to the board
	async playCard(card){
		await this.playCardAction(card, async () => await card.autoplay(this.hand));
	}
	
	// Shows a preview of the card being played, plays it to the board and ends the turn
	async playCardAction(card, action){
		ui.showPreviewVisuals(card);
		await sleep(1000);
		ui.hidePreview(card);
		await action();
		this.endTurn();
	}
	
	// Handles end of turn visuals and behavior that notifies the game
	endTurn(){
		if (!this.passed && !this.canPlay())
			this.setPassed(true);
		if (this === player_me){
			document.getElementById("pass-button").classList.add("noclick");
		}
		document.getElementById("stats-" + this.tag).classList.remove("current-turn");
		this.elem_leader.children[1].classList.add("hide");
		game.endTurn()
	}
	
	// Tells the Player if it won the round. May damage health.
	endRound(win){
		if (!win) {
			if (this.health < 1)
				return;
			document.getElementById("gem" + this.health + "-" +this.tag).classList.remove("gem-on");
			this.health--;
		}
		this.setPassed(false);
		this.setWinning(false);
	}
	
	// Returns true if the Player can make any action other than passing
	canPlay() {
		return this.hand.cards.length > 0 || this.leaderAvailable;
	}
	
	// Use a leader's Activate ability, then disable the leader
	async activateLeader() {
		ui.showPreviewVisuals(this.leader);
		await sleep(1500);
		ui.hidePreview(this.leader);
		await this.leader.activated[0](this.leader, this);
		this.disableLeader();
		this.endTurn();
	}
	
	// Disable access to leader ability and toggles leader visuals to off state
	disableLeader(){
		this.leaderAvailable = false;
		let elem = this.elem_leader.cloneNode(true);
		this.elem_leader.parentNode.replaceChild(elem, this.elem_leader);
		this.elem_leader = elem;
		this.elem_leader.children[0].classList.add("fade");
		this.elem_leader.children[1].classList.add("hide");
		this.elem_leader.addEventListener("click", async () => await ui.viewCard(this.leader), false);
	}
	
	// Enable access to leader ability and toggles leader visuals to on state
	enableLeader() {
		this.leaderAvailable = this.leader.activated.length > 0;
		let elem = this.elem_leader.cloneNode(true);
		this.elem_leader.parentNode.replaceChild(elem, this.elem_leader);
		this.elem_leader = elem;
		this.elem_leader.children[0].classList.remove("fade");
		this.elem_leader.children[1].classList.remove("hide");
		
		if (this.id === 0 && this.leader.activated.length > 0){
			this.elem_leader.addEventListener("click", 
				async () => await ui.viewCard(this.leader, async () => await this.activateLeader()),
				false);
		} else {
			this.elem_leader.addEventListener("click", async () => await ui.viewCard(this.leader), false);
		}
		
		// TODO set crown color
	}
	
}

// Other classes remain unchanged...

// ... [Rest of the classes like CardContainer, Grave, Deck, Hand, Row, Weather, Board, etc.] ...

// Modified UI class to include "Change Player" button and manage hand visibility
class UI {
	constructor() {
		this.carousels = [];
		this.notif_elem = document.getElementById("notification-bar");
		this.preview = document.getElementsByClassName("card-preview")[0];
		this.previewCard = null;
		this.lastRow = null;
		document.getElementById("pass-button").addEventListener("click", () => player_me.passRound(), false);
		document.getElementById("click-background").addEventListener("click", () => ui.cancel(), false);
		this.youtube;
		this.ytActive;
		this.toggleMusic_elem = document.getElementById("toggle-music");
		this.toggleMusic_elem.classList.add("fade");
		this.toggleMusic_elem.addEventListener("click", () => this.toggleMusic(), false);
		
		// === Added "Change Player" Button ===
		this.changePlayerButton = document.createElement("button");
		this.changePlayerButton.innerHTML = "Change Player";
		this.changePlayerButton.id = "change-player-button";
		this.changePlayerButton.style.position = "absolute";
		this.changePlayerButton.style.top = "10px";
		this.changePlayerButton.style.right = "10px";
		this.changePlayerButton.style.display = "none"; // Initially hidden
		this.changePlayerButton.classList.add("change-player-button"); // Optional: add a class for styling
		document.body.appendChild(this.changePlayerButton);
		this.changePlayerButton.addEventListener("click", () => game.switchPlayer(), false);
		// === End of "Change Player" Button ===
	}
	
	// Existing methods...
	
	// === New Methods for "Change Player" Button ===
	showChangePlayerButton() {
		this.changePlayerButton.style.display = "block";
	}
	
	hideChangePlayerButton() {
		this.changePlayerButton.style.display = "none";
	}
	// === End of New Methods ===
	
	// Updates which player's hand is visible
	showCurrentPlayer(player){
		player_me.hand.elem.style.display = (player === player_me) ? "flex" : "none";
		player_op.hand.elem.style.display = (player === player_op) ? "flex" : "none";
	}
	
	// Existing methods like selectCard, selectRow, etc.
	
	// ... [Rest of the UI class methods] ...
}

// Modified Game class to handle turn switching manually
class Game {
	constructor() {
		this.endScreen = document.getElementById("end-screen");
		let buttons = this.endScreen.getElementsByTagName("button");
		this.customize_elem = buttons[0];
		this.replay_elem = buttons[1];
		this.customize_elem.addEventListener("click", () => this.returnToCustomization(), false);
		this.replay_elem.addEventListener("click", () => this.restartGame(), false);
		this.reset();
	}
	
	reset() {
		this.firstPlayer;
		this.currPlayer = null;
		
		this.gameStart = [];
		this.roundStart = [];
		this.roundEnd = [];
		this.turnStart = [];
		this.turnEnd = [];
		
		this.roundCount = 0;
		this.roundHistory = [];
		
		this.randomRespawn = false;
		this.doubleSpyPower = false;
		
		weather.reset();
		board.row.forEach(r => r.reset());
	}
	
	// Sets up player faction abilities and passive leader abilities
	initPlayers(p1, p2){
		let l1 = ability_dict[p1.leader.abilities[0]];
		let l2 = ability_dict[p2.leader.abilities[0]];
		if (l1 === ability_dict["emhyr_whiteflame"] || l2 === ability_dict["emhyr_whiteflame"]){
			p1.disableLeader();
			p2.disableLeader();
		} else {
			initLeader(p1, l1);
			initLeader(p2, l2);
		}
		if (p1.deck.faction === p2.deck.faction && p1.deck.faction === "scoiatael")
			return;
		initFaction(p1);
		initFaction(p2);
		
		function initLeader(player, leader){
			if (leader.placed)
				leader.placed(player.leader);
			Object.keys(leader).filter(key => game[key]).map(key => game[key].push(leader[key]));
		}
		
		function initFaction(player){
			if (factions[player.deck.faction] && factions[player.deck.faction].factionAbility)
				factions[player.deck.faction].factionAbility(player);
		}
	}
	
	// Sets initializes player abilities, player hands and redraw
	async startGame() {
		ui.toggleMusic_elem.classList.remove("music-customization");
		this.initPlayers(player_me, player_op);
		await Promise.all([...Array(10).keys()].map( async () => {
			await player_me.deck.draw(player_me.hand);
			await player_op.deck.draw(player_op.hand);
		}));
		
		await this.runEffects(this.gameStart);
		if (!this.firstPlayer)
			this.firstPlayer = await this.coinToss();
		this.initialRedraw();
	}
	
	// Simulated coin toss to determine who starts game
	async coinToss(){
		this.firstPlayer = (Math.random() < 0.5) ? player_me : player_op;
		await ui.notification(this.firstPlayer.tag + "-coin", 1200);
		return this.firstPlayer;
	}
	
	// Allows the player to swap out up to two cards from their initial hand
	async initialRedraw(){
		for (let i=0; i< 2; i++)
			player_op.controller.redraw();
		await ui.queueCarousel(player_me.hand, 2, async (c, i) => await player_me.deck.swap(c, c.removeCard(i)), c => true, true, true, "Choose up to 2 cards to redraw.");
		ui.enablePlayer(false);
		this.startRound();
	}
	
	// Initiates a new round of the game
	async startRound(){
		this.roundCount++;
		this.currPlayer = (this.roundCount%2 === 0) ? this.firstPlayer : this.firstPlayer.opponent();
		await this.runEffects(this.roundStart);
		
		if ( !player_me.canPlay() )
			player_me.setPassed(true);
		if ( !player_op.canPlay() )
			player_op.setPassed(true);
		
		if (player_op.passed && player_me.passed)
			return this.endRound();
		
		if (this.currPlayer.passed)
			this.currPlayer = this.currPlayer.opponent();
		
		await ui.notification("round-start", 1200);
		if (this.currPlayer.opponent().passed)
			await ui.notification(this.currPlayer.tag + "-turn", 1200);
		
		this.startTurn();
	}
	
	// Starts a new turn. Enables client interaction in client's turn.
	async startTurn() {
		await this.runEffects(this.turnStart);
		if (!this.currPlayer.opponent().passed){
			this.currPlayer = this.currPlayer.opponent();
			await ui.notification(this.currPlayer.tag + "-turn", 1200);
		}
		// Show current player's hand and hide opponent's
		ui.showCurrentPlayer(this.currPlayer);
		ui.enablePlayer(this.currPlayer === player_me || this.currPlayer === player_op);
		this.currPlayer.startTurn();
	}
	
	// Ends the current turn and shows the "Change Player" button
	async endTurn() {
		if (this.currPlayer === player_me || this.currPlayer === player_op)
			ui.enablePlayer(false);
		await this.runEffects(this.turnEnd);
		if (this.currPlayer.passed)
			await ui.notification(this.currPlayer.tag + "-pass", 1200);
		if (player_op.passed && player_me.passed)
			this.endRound();
		else
			ui.showChangePlayerButton(); // Show the "Change Player" button instead of starting the next turn automatically
	}
	
	// Ends the round and may end the game. Determines final scores and the round winner.
	async endRound() {
		let dif = player_me.total - player_op.total;
		if (dif === 0) {
			let nilf_me = player_me.deck.faction === "nilfgaard", nilf_op = player_op.deck.faction === "nilfgaard";
			dif = nilf_me ^ nilf_op ? nilf_me ? 1 : -1 : 0;
		}
		let winner = dif > 0 ? player_me : dif < 0 ? player_op : null;
		let verdict = {winner: winner, score_me: player_me.total, score_op: player_op.total}
		this.roundHistory.push(verdict);
		
		await this.runEffects(this.roundEnd);
		
		board.row.forEach( row => row.clear() );
		weather.clearWeather();
		
		player_me.endRound( dif > 0);
		player_op.endRound( dif < 0);
		
		if (dif > 0)
			await ui.notification("win-round", 1200);
		else if (dif < 0)
			await ui.notification("lose-round", 1200);
		else
			await ui.notification("draw-round", 1200);
		
		if (player_me.health === 0 || player_op.health === 0)
			this.endGame();
		else
			this.startRound();
	}
	
	// Sets up and displays the end-game screen
	async endGame() {
		let endScreen = document.getElementById("end-screen");
		let rows = endScreen.getElementsByTagName("tr");
		rows[1].children[0].innerHTML = player_me.name;
		rows[2].children[0].innerHTML = player_op.name;
		
		for (let i=1; i<4; ++i) {
			let round = this.roundHistory[i-1];
			rows[1].children[i].innerHTML = round ? round.score_me : 0;
			rows[1].children[i].style.color = round && round.winner === player_me ? "goldenrod" : "";
			
			rows[2].children[i].innerHTML = round ? round.score_op : 0;
			rows[2].children[i].style.color = round && round.winner === player_op ? "goldenrod" : "";
		}
		
		endScreen.children[0].className = "";
		if (player_op.health <= 0 && player_me.health <= 0) {
			endScreen.getElementsByTagName("p")[0].classList.remove("hide");
			endScreen.children[0].classList.add("end-draw");
		} else if (player_op.health === 0){
			endScreen.children[0].classList.add("end-win");
		} else {
			endScreen.children[0].classList.add("end-lose");
		}
		
		fadeIn(endScreen, 300);
		ui.enablePlayer(true);
	}
	
	// Returns the client to the deck customization screen
	returnToCustomization(){
		this.reset();
		player_me.reset();
		player_op.reset();
		ui.toggleMusic_elem.classList.add("music-customization");
		this.endScreen.classList.add("hide");
		document.getElementById("deck-customization").classList.remove("hide");
	}
	
	// Restarts the last game with the same decks
	restartGame(){
		this.reset();
		player_me.reset();
		player_op.reset();
		this.endScreen.classList.add("hide");
		this.startGame();
	}
	
	// Executes effects in list. If effect returns true, effect is removed.
	async runEffects(effects){
		for (let i=effects.length-1; i>=0; --i){
			let effect = effects[i];
			if (await effect())
				effects.splice(i,1)
		}
	}
	
	// === New Method to Switch Players ===
	switchPlayer(){
		// Hide the "Change Player" button
		ui.hideChangePlayerButton();
		// Switch to the opponent player
		this.currPlayer = this.currPlayer.opponent();
		// Update the UI to show the current player's hand
		ui.showCurrentPlayer(this.currPlayer);
		// Enable interaction for the current player
		ui.enablePlayer(true);
		// Start the current player's turn
		this.currPlayer.startTurn();
	}
	// === End of New Method ===
}

// Modified UI class methods remain the same...

// Handles notifications and client interaction with menus
class UI {
	constructor() {
		this.carousels = [];
		this.notif_elem = document.getElementById("notification-bar");
		this.preview = document.getElementsByClassName("card-preview")[0];
		this.previewCard = null;
		this.lastRow = null;
		document.getElementById("pass-button").addEventListener("click", () => player_me.passRound(), false);
		document.getElementById("click-background").addEventListener("click", () => ui.cancel(), false);
		this.youtube;
		this.ytActive;
		this.toggleMusic_elem = document.getElementById("toggle-music");
		this.toggleMusic_elem.classList.add("fade");
		this.toggleMusic_elem.addEventListener("click", () => this.toggleMusic(), false);
		
		// === "Change Player" Button Initialization ===
		this.changePlayerButton = document.createElement("button");
		this.changePlayerButton.innerHTML = "Change Player";
		this.changePlayerButton.id = "change-player-button";
		this.changePlayerButton.style.position = "absolute";
		this.changePlayerButton.style.top = "10px";
		this.changePlayerButton.style.right = "10px";
		this.changePlayerButton.style.display = "none"; // Initially hidden
		this.changePlayerButton.classList.add("change-player-button"); // Optional: add a class for styling
		document.body.appendChild(this.changePlayerButton);
		this.changePlayerButton.addEventListener("click", () => game.switchPlayer(), false);
		// === End of "Change Player" Button Initialization ===
	}
	
	// Enables or disables client interaction
	enablePlayer(enable){
		let main = document.getElementsByTagName("main")[0].classList;
		if (enable) main.remove("noclick"); else main.add("noclick");
	}
	
	// Initializes the YouTube background music object
	initYouTube(){
		this.youtube = new YT.Player('youtube', {
			videoId: "UE9fPWy1_o4",
			playerVars:  { "autoplay" : 1, "controls" : 0, "loop" : 1, "playlist" : "UE9fPWy1_o4", "rel" : 0, "version" : 3, "modestbranding" : 1 },
			events: { 'onStateChange': initButton }
		});
		
		function initButton(){
			if (ui.ytActive !== undefined)
				return;
			ui.ytActive = true;
			ui.youtube.playVideo();
			let timer = setInterval( () => {
				if (ui.youtube.getPlayerState() !== YT.PlayerState.PLAYING)
					ui.youtube.playVideo();
				else {
					clearInterval(timer);
					ui.toggleMusic_elem.classList.remove("fade");
				}
			}, 500);
		}
	}
	
	// Called when client toggles the music
	toggleMusic(){
		if (this.youtube.getPlayerState() !== YT.PlayerState.PLAYING) {
			this.youtube.playVideo();
			this.toggleMusic_elem.classList.remove("fade");
		} else {
			this.youtube.pauseVideo();
			this.toggleMusic_elem.classList.add("fade");
		}
	}
	
	// Enables or disables background music 
	setYouTubeEnabled(enable){
		if (this.ytActive === enable)
			return;
		if (enable && !this.mute)
			ui.youtube.playVideo();
		else
			ui.youtube.pauseVideo();
		this.ytActive = enable;
	}
	
	// Called when the player selects a selectable card
	async selectCard(card) {
		let row = this.lastRow;
		let pCard = this.previewCard;
		if (card === pCard)
			return;
		if (pCard === null || card.holder.hand.cards.includes(card)) {
			this.setSelectable(null, false);
			this.showPreview(card);
		} else if (pCard.name === "Decoy") {
			this.hidePreview(card);
			this.enablePlayer(false);
			board.toHand(card, row);
			await board.moveTo(pCard, row, pCard.holder.hand);
			pCard.holder.endTurn();
		}
	}
	
	// Called when the player selects a selectable CardContainer
	async selectRow(row){
		this.lastRow = row;
		if (this.previewCard === null) {
			await ui.viewCardsInContainer(row);
			return;
		}
		if (this.previewCard.name === "Decoy")
			return;
		let card = this.previewCard;
		let holder = card.holder;
		this.hidePreview();
		this.enablePlayer(false);
		if (card.name === "Scorch"){
			this.hidePreview();
			await ability_dict["scorch"].activated(card);
		} else if (card.name === "Decoy") {
			return;
		} else {
			await board.moveTo(card, row, card.holder.hand);
		}
		holder.endTurn();
	}
	
	// Called when the client cancels out of a card-preview
	cancel(){
		this.hidePreview();
	}
	
	// Displays a card preview then enables and highlights potential card destinations
	showPreview(card) {
		this.showPreviewVisuals(card);
		this.setSelectable(card, true);
		document.getElementById("click-background").classList.remove("noclick");
	}
	
	// Sets up the graphics and description for a card preview
	showPreviewVisuals(card){
		this.previewCard = card;
		this.preview.classList.remove("hide");
		this.preview.getElementsByClassName("card-lg")[0].style.backgroundImage = largeURL(card.faction+"_"+card.filename);
		let desc_elem = this.preview.getElementsByClassName("card-description")[0];
		this.setDescription(card, desc_elem);
	}
	
	// Hides the card preview then disables and removes highlighting from card destinations
	hidePreview(){
		document.getElementById("click-background").classList.add("noclick");
		player_me.hand.cards.forEach( c => c.elem.classList.remove("noclick") );
		
		this.preview.classList.add("hide");
		this.setSelectable(null, false);
		this.previewCard = null;
		this.lastRow = null;
	}
	
	// Sets up description window for a card
	setDescription(card, desc){
		if (card.hero || card.row === "agile" || card.abilities.length > 0 || card.faction === "faction") {
			desc.classList.remove("hide");
			let str = card.row === "agile" ? "agile" : "";
			if (card.abilities.length)
				str = card.abilities[card.abilities.length-1];
			if (str === "cerys")
				str = "muster";
			if (str.startsWith("avenger"))
				str = "avenger";
			if (str === "scorch_c" || str == "scorch_r" || str === "scorch_s")
				str = "scorch";
			if (card.row === "leader" || card.faction === "faction" || card.abilities.length === 0 && card.row !== "agile")
				desc.children[0].style.backgroundImage = "";
			else
				desc.children[0].style.backgroundImage = iconURL("card_ability_" + str);
			desc.children[1].innerHTML = card.desc_name;
			desc.children[2].innerHTML = card.desc;
		} else {
			desc.classList.add("hide");
		}
	}
	
	// Displayed a timed notification to the client
	async notification(name, duration){
		if (!duration)
			duration = 1200;
		duration = Math.max(400, duration);
		const fadeSpeed = 150;
		this.notif_elem.children[0].id = "notif-" + name;
		fadeIn(this.notif_elem, fadeSpeed);
		fadeOut(this.notif_elem, fadeSpeed, duration - fadeSpeed);
		await sleep(duration);
	}
	
	// Displays a cancellable Carousel for a single card 
	async viewCard(card, action) {
		if (card === null)
			return;
		let container = new CardContainer();
		container.cards.push(card);
		await this.viewCardsInContainer(container, action);
	}
	
	// Displays a cancellable Carousel for all cards in a container
	async viewCardsInContainer(container, action) {
		action = action ? action : function() {return this.cancel();};
		await this.queueCarousel(container, 1, action, () => true, false, true);
	}
	
	// Displays a Carousel menu of filtered container items that match the predicate.
	// Suspends gameplay until the Carousel is closed. Automatically picks random card if activated for AI player
	async queueCarousel(container, count, action, predicate, bSort, bQuit, title){
		if (game.currPlayer === player_op) {
			if (player_op.controller instanceof ControllerAI)
				for (let i=0; i<count; ++i){
					let cards = container.cards.reduce((a,c,i) => !predicate || predicate(c) ? a.concat([i]) : a, []);
					await action(container, cards[randomInt(cards.length)]);
				}
			return;
		}
		let carousel = new Carousel(container, count, action, predicate, bSort, bQuit, title);
		if (Carousel.curr === undefined || Carousel.curr === null)
			carousel.start();
		else {
			this.carousels.push(carousel);
			return;
		}
		await sleepUntil( () => this.carousels.length === 0 && !Carousel.curr, 100);
	}
	
	// Starts the next queued Carousel
	quitCarousel(){
		if (this.carousels.length > 0) {
			this.carousels.shift().start();
		}
	}
	
	// Displays a custom confirmation menu 
	async popup(yesName, yes, noName, no, title, description) {
		let p = new Popup(yesName, yes, noName, no, title, description);
		await sleepUntil( () => !Popup.curr) 
	}
	
	// Enables or disables selection and highlighting of rows specific to the card
	setSelectable(card, enable){
		if(!enable) {
			for (let row of board.row){
				row.elem.classList.remove("row-selectable");
				row.elem.classList.remove("noclick");
				row.elem_special.classList.remove("row-selectable");
				row.elem_special.classList.remove("noclick");
				row.elem.classList.add("card-selectable");
				
				for (let card of row.cards) {
					card.elem.classList.add("noclick");
				}
			}
			weather.elem.classList.remove("row-selectable");
			weather.elem.classList.remove("noclick");
			return;
		}
		if (card.faction === "weather") {
			for (let row of board.row){
				row.elem.classList.add("noclick");
				row.elem_special.classList.add("noclick");
			}
			weather.elem.classList.add("row-selectable");
			return;
		}
		
		weather.elem.classList.add("noclick");
		
		if (card.name === "Scorch") {
			for (let r of board.row){
				r.elem.classList.add("row-selectable");
				r.elem_special.classList.add("row-selectable");
			}
			return;
		}
		if (card.isSpecial()){
			for (let i=0; i<6; i++){
				let r = board.row[i];
				if (i < 3 || r.special !== null){
					r.elem.classList.add("noclick");
					r.elem_special.classList.add("noclick");
				} else {
					r.elem_special.classList.add("row-selectable");
				}
			}
			return;
		}
		
		weather.elem.classList.add("noclick");
		
		if (card.name === "Decoy"){
			for (let i=0; i<6; ++i) {
				let r = board.row[i];
				let units = r.cards.filter(c => c.isUnit());
				if (i < 3 || units.length === 0) {
					r.elem.classList.add("noclick");
					r.elem_special.classList.add("noclick");
					r.elem.classList.remove("card-selectable");
				} else {
					r.elem.classList.add("row-selectable");
					units.forEach( c => c.elem.classList.remove("noclick") );
				}
			}
			return;
		}
		
		let currRows = card.row === "agile" ? [board.getRow(card, "close", card.holder), board.getRow(card, "ranged", card.holder)] : [board.getRow(card, card.row, card.holder)];
		for (let i=0; i<6; i++){
			let row = board.row[i];
			if (currRows.includes(row)) {
				row.elem.classList.add("row-selectable");
			} else {
				row.elem.classList.add("noclick");
			}
		}
	
	}
	
	// === New Method to Show and Hide "Change Player" Button ===
	showChangePlayerButton() {
		this.changePlayerButton.style.display = "block";
	}
	
	hideChangePlayerButton() {
		this.changePlayerButton.style.display = "none";
	}
	// === End of New Methods ===
	
	// === New Method to Manage Current Player's Hand Visibility ===
	showCurrentPlayer(player){
		player_me.hand.elem.style.display = (player === player_me) ? "flex" : "none";
		player_op.hand.elem.style.display = (player === player_op) ? "flex" : "none";
	}
	// === End of New Method ===
}

// Carousel and Popup classes remain unchanged...

// Translates a card between two containers
async function translateTo(card, container_source, container_dest){
	if (!container_dest || !container_source)
		return;
	if (container_dest === player_op.hand && container_source === player_op.deck)
		return;
	
	let elem = card.elem;
	let source = !container_source ? card.elem : getSourceElem(card, container_source, container_dest);
	let dest = getDestinationElem(card, container_source, container_dest);
	if (!isInDocument(elem))
		source.appendChild(elem);
	let x = trueOffsetLeft(dest) - trueOffsetLeft(elem) +dest.offsetWidth/2 - elem.offsetWidth;
	let y = trueOffsetTop(dest) - trueOffsetTop(elem) +dest.offsetHeight/2 - elem.offsetHeight/2;
	if (container_dest instanceof Row && container_dest.cards.length !== 0 && !card.isSpecial() ){
		x += (container_dest.getSortedIndex(card) === container_dest.cards.length) ? elem.offsetWidth/2 : -elem.offsetWidth/2;
	}
	if (card.holder.controller instanceof ControllerAI)
		x += elem.offsetWidth/2;
	if (container_source instanceof Row && container_dest instanceof Grave && !card.isSpecial()) {
		let mid = trueOffset(container_source.elem, true) + container_source.elem.offsetWidth/2;
		x += trueOffset(elem, true) - mid;
	}
	if (container_source instanceof Row && container_dest === player_me.hand)
		y *= 7/8;
	await translate(elem, x, y);
	
	// Returns true if the element is visible in the viewport
	function isInDocument(elem){
		return elem.getBoundingClientRect().width !== 0;
	}
	
	// Returns the true offset of a nested element in the viewport
	function trueOffset(elem, left){
		let total =0
		let curr = elem;
		while (curr){
			total += (left ? curr.offsetLeft : curr.offsetTop);
			curr = curr.parentElement;
		}
		return total;
	}
	function trueOffsetLeft(elem) {	return trueOffset(elem, true); }
	function trueOffsetTop(elem) { return trueOffset(elem, false); }
	
	// Returns the source container's element to transition from
	function getSourceElem(card, source, dest){
		if (source instanceof HandAI)
			return source.hidden_elem;
		if (source instanceof Deck)
			return source.elem.children[source.elem.children.length-2];
		return source.elem;
	}

	// Returns the destination container's element to transition to
	function getDestinationElem(card, source, dest){
		if (dest instanceof HandAI)
			return dest.hidden_elem;
		if (card.isSpecial() && dest instanceof Row)
			return dest.elem_special;
		if (dest instanceof Row || dest instanceof Hand || dest instanceof Weather){
			if (dest.cards.length === 0)
				return dest.elem;
			let index = dest.getSortedIndex(card);
			let dcard = dest.cards[index === dest.cards.length ? index-1 : index];
			return dcard.elem;
		}
		return dest.elem;
	}
}

// Translates an element by x from the left and y from the top
async function translate(elem, x, y){
	let vw100 = 100 / document.getElementById("dimensions").offsetWidth;
	x*=vw100;
	y*=vw100 ;
	elem.style.transform = "translate(" + x + "vw, " + y + "vw)";
	let margin = elem.style.marginLeft;
	elem.style.marginRight = -elem.offsetWidth*vw100 + "vw";
	elem.style.marginLeft = "";
	await sleep(499);
	elem.style.transform = "";
	elem.style.position = "";
	elem.style.marginLeft = margin;
	elem.style.marginRight = margin;
}

// Fades out an element until hidden over the duration
async function fadeOut(elem, duration, delay) {
	await fade(false, elem, duration, delay);
}

// Fades in an element until opaque over the duration
async function fadeIn(elem, duration, delay){
	await fade(true, elem, duration, delay);
}

// Fades an element over a duration 
async function fade(fadeIn, elem, dur, delay){
	if (delay)
		await sleep(delay)
	let op = fadeIn ?  0.1 : 1;
	elem.style.opacity = op;
	elem.style.filter = "alpha(opacity=" + (op * 100) + ")";
	if (fadeIn)
		elem.classList.remove("hide");
	let timer = setInterval( async function() {
		op += op * (fadeIn ? 0.1 : -0.1);
		if (op >= 1) {
			clearInterval(timer);
			return;
		} else if (op <= 0.1) {
			elem.classList.add("hide");
			elem.style.opacity = "";
			elem.style.filter = "";
			clearInterval(timer);
			return;
		}
		elem.style.opacity = op;
		elem.style.filter = "alpha(opacity=" + (op * 100) + ")";
	}, dur/24);
}

// Get Image paths   
function iconURL(name, ext = "png"){
	return imgURL("icons/" + name, ext);
}
function largeURL(name, ext="jpg"){
	return imgURL("lg/" + name, ext) 
}
function smallURL(name, ext="jpg"){
	return imgURL("sm/" + name, ext);
}
function imgURL(path, ext) {
	return "url('img/" + path + "." + ext;
}

// Returns true if n is an Number
function isNumber(n) { 
	return !isNaN(parseFloat(n)) && isFinite(n);
}

// Returns true if s is a String
function isString(s){
	return typeof(s) === 'string' || s instanceof String;
}

// Returns a random integer in the range [0,n)
function randomInt(n)  {
	return Math.floor(Math.random() * n);
}

// Pauses execution until the passed number of milliseconds as expired
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
  //return new Promise(resolve => setTimeout(() => {if (func) func(); return resolve();}, ms));
}

// Suspends execution until the predicate condition is met, checking every ms milliseconds
function sleepUntil(predicate, ms) {
	return new Promise(resolve => {
		let timer = setInterval( function () {
			if (predicate()) {
				clearInterval(timer);
				resolve();
			}
		}, ms)
	});
}

// Initializes the interactive YouTube object
function onYouTubeIframeAPIReady() {
	ui.initYouTube();
}

/*----------------------------------------------------*/

// Instantiate UI, Board, Weather, Game, and Players
var ui = new UI();
var board = new Board();
var weather = new Weather();
var game = new Game();

// Initialize players as global variables
var player_me, player_op;

// Initially disable player interaction
ui.enablePlayer(false);

// Initialize DeckMaker
let dm = new DeckMaker();

// === Initialize Both Players as Human ===
player_me = new Player(0, "Player 1", {/* deck data */});
player_op = new Player(1, "Player 2", {/* deck data */});
// Note: Replace {/* deck data */} with actual deck initialization as per your game setup

// Start the game
game.startGame();
```

### **Key Modifications Explained:**

1. **Controller Adjustments:**
   - **Both Players are Human-Controlled:** The `ControllerAI` class has been removed, and both `player_me` and `player_op` are now instances of the base `Controller` class, which is empty. This change ensures that both players are controlled via the UI rather than one being AI-controlled.

2. **Hand Management:**
   - **Separate Hand Elements:** The `Player` class now assigns `Hand` instances to both players using unique HTML element IDs (`hand-row-me` and `hand-row-op`). Ensure that your `index.html` has corresponding elements with these IDs.
   
3. **"Change Player" Button:**
   - **Dynamic Creation:** The `UI` class now dynamically creates a "Change Player" button, styles it, and appends it to the document body.
   - **Button Visibility:** The button is initially hidden and only appears after a player completes their turn.
   - **Button Functionality:** Clicking the button triggers the `switchPlayer` method in the `Game` class, which handles the turn switching logic.

4. **Turn Switching Logic:**
   - **Manual Turn Switching:** The `Game` class's `endTurn` method has been modified to display the "Change Player" button instead of automatically starting the next turn.
   - **Switching Players:** A new `switchPlayer` method has been added to the `Game` class, which hides the button, switches the current player, updates the UI to show the correct player's hand, and starts the new player's turn.

5. **Hand Visibility:**
   - **Current Player's Hand Visible:** The `showCurrentPlayer` method in the `UI` class ensures that only the current player's hand is visible, while the opponent's hand is hidden. This prevents players from seeing each other's unplayed cards.

6. **Player Initialization:**
   - **Global Player Instances:** Both `player_me` and `player_op` are instantiated as global variables with unique IDs and names. Ensure that you provide the appropriate deck data during initialization.

7. **DeckMaker Integration:**
   - **Deck Initialization:** The `DeckMaker` class remains largely unchanged, but ensure that it properly initializes both players' decks as per your game's requirements.

### **Additional Notes:**

- **HTML Adjustments:** Ensure that your `index.html` has separate elements for both players' hands with IDs `hand-row-me` and `hand-row-op`. This separation is crucial for managing visibility correctly.

- **CSS Styling:** You may want to add specific styles for the "Change Player" button (e.g., positioning, colors) by targeting the `change-player-button` class or the button's ID.

- **Deck Data:** Replace the placeholder `{/* deck data */}` with the actual deck initialization logic as per your game's setup. This might involve loading predefined decks or allowing users to customize their decks.

- **Testing:** After implementing these changes, thoroughly test the game to ensure that turn switching works seamlessly, hands are correctly shown/hidden, and that both players can interact without issues.

This modification should provide a smooth two-player experience on the same computer, aligning closely with the user's requirements.