<script>
	import { onMount } from 'svelte';
	import { validateSencode } from '@sudoku/sencode';
	import game from '@sudoku/game';
	import { initHistoryFromLive } from '@sudoku/stores/history/board_state';
	import { modal } from '@sudoku/stores/modal';
	import { gameWon } from '@sudoku/stores/game';
	import { userGrid, invalidCells } from '@sudoku/stores/grid';
	import { get } from 'svelte/store';
	import Board from './components/Board/index.svelte';
	import Controls from './components/Controls/index.svelte';
	import Header from './components/Header/index.svelte';
	import Modal from './components/Modal/index.svelte';

	gameWon.subscribe(won => {
		if (won) {
			// Debug info to help trace why gameWon fired during undo/redo
			const $userGrid = get(userGrid);
			const emptyCount = $userGrid.flat().filter(v => v === 0).length;
			console.debug('[gameWon] fired ->', { emptyCount, invalidCells: get(invalidCells).length, stack: (new Error()).stack });
			game.pause();
			modal.show('gameover');
		}
	});

	onMount(() => {
		let hash = location.hash;

		if (hash.startsWith('#')) {
			hash = hash.slice(1);
		}

		let sencode;
		if (validateSencode(hash)) {
			sencode = hash;
		}

		// Ensure history root reflects current live grids on app start
		try { initHistoryFromLive(); } catch (e) {}

		modal.show('welcome', { onHide: game.resume, sencode });
	});
</script>

<!-- Timer, Menu, etc. -->
<header>
	<Header />
</header>

<!-- Sudoku Field -->
<section>
	<Board />
</section>

<!-- Keyboard -->
<footer>
	<Controls />
</footer>

<Modal />

<style global>
	@import "./styles/global.css";
</style>