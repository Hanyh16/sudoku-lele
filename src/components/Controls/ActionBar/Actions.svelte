<script>
	import { candidates } from '@sudoku/stores/candidates';
	import { userGrid, strategyGrid } from '@sudoku/stores/grid';
	import { cursor } from '@sudoku/stores/cursor';
	import { hints } from '@sudoku/stores/hints';
	import { notes } from '@sudoku/stores/notes';
	import { settings } from '@sudoku/stores/settings';
	import { keyboardDisabled } from '@sudoku/stores/keyboard';
	import { gamePaused } from '@sudoku/stores/game';
	import { strategyManager } from '@sudoku/strategy/strategyManager';
	import { historyManager } from '@sudoku/stores/history/history';
	import { get } from 'svelte/store';

	// 订阅时间步与分支次数与撤销重做计数，用于按钮禁用状态
	const timeStepStore = strategyGrid.getTimeStep();
	const branchBackTimesStore = historyManager.getBranchBackTimes();
	const canUndoStore = historyManager.getCanUndo();
	const canRedoStore = historyManager.getCanRedo();
	const canBacktrackStore = historyManager.getCanBacktrack();

	$: canUndo = $canUndoStore;
	$: canRedo = $canRedoStore;
	$: canBacktrack = $canBacktrackStore;

	// 调试：打印这三个 store 的当前值以及解析后的布尔值，便于排查按钮禁用逻辑
	$: console.debug('[Actions] $canUndoStore, $canRedoStore, $canBacktrackStore ->', $canUndoStore, $canRedoStore, $canBacktrackStore);
	$: console.debug('[Actions] resolved booleans ->', { canUndo, canRedo, canBacktrack });

	$: hintsAvailable = $hints > 0;
	// $: hintsAvailable = true;

	function handleHint() {
		console.debug('[hint] button clicked');
		if (hintsAvailable) {
			console.debug('[hint] requested');
			if ($candidates.hasOwnProperty($cursor.x + ',' + $cursor.y)) {
				candidates.clear($cursor);
			}

			// 记录当前时间步，用于通过命令推进到下一步
			const prevTimeStep = get(strategyGrid.getTimeStep());
			// 先应用策略以更新当前步的候选/相对位置/单候选 explore
			const strategyApplyCell = strategyManager.apply(get(strategyGrid.getStrategyGrid()));
			// strategyApplyCell.forEach(pos => strategyGrid.setCurrentCell(pos));
      		// strategyGrid.updateCellCandidates();
			// 更新策略管理器状态
			if (strategyApplyCell.length > 0) {
				console.debug('[hint] applied', {
					timeStep: get(strategyGrid.getTimeStep()),
					positions: strategyApplyCell,
				});
				strategyManager.getIsUsingStrategy().set(true);
				const isSingle = get(strategyManager.getIsGenerateSingleCandidate());
				console.debug('[hint] flags', { isSingle });
				const exploreValues = isSingle
					? strategyApplyCell.map(pos => get(strategyGrid.getStrategyGrid())[pos.y][pos.x].explore)
					: [];
				const action = historyManager.createHintAction({
					timeStepBefore: prevTimeStep,
					positions: strategyApplyCell,
					isSingle: isSingle,
					values: exploreValues,
				});
				const ok = action.apply();
				if (ok) {
					historyManager.recordAction(action);
					if (isSingle) hints.useHint();
					console.debug('[hint] committed', {
						nextTimeStep: get(strategyGrid.getTimeStep()),
						positions: strategyApplyCell,
						branchCount: get(historyManager.getBranchBackTimes()),
					});
				}

			} else {
				strategyManager.getIsUsingStrategy().set(false);
			}
		}
	}
</script>

<div class="action-buttons space-x-3">

	<button class="btn btn-round btn-badge" disabled={$gamePaused || !canBacktrack} on:click={() => historyManager.backtrackWithRecord()} title="Backtrack ({$branchBackTimesStore})">
		<svg class="icon-outline" fill="none" height="24" stroke="currentColor" stroke-linecap="round"
             stroke-linejoin="round"
             stroke-width="2" viewBox="0 0 24 24" width="24" xmlns="http://www.w3.org/2000/svg">
            <polyline points="1 4 1 10 7 10"></polyline>
            <path d="M3.51 15a9 9 0 1 0 2.13-9.36L1 10"></path>
		</svg>

		{#if canBacktrack}
			<span class="badge" class:badge-primary={canBacktrack}>{$branchBackTimesStore}</span>
		{/if}
	</button>
    
    <button class="btn btn-round" disabled={$gamePaused || !canUndo} on:click={() => historyManager.undoStep()} title="Undo">
		<svg class="icon-outline" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
			<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 10h10a8 8 0 018 8v2M3 10l6 6m-6-6l6-6" />
		</svg>
	</button>

	<button class="btn btn-round" disabled={$gamePaused || !canRedo} on:click={() => historyManager.redoStep()} title="Redo">
		<svg class="icon-outline" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
			<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 10h-10a8 8 90 00-8 8v2M21 10l-6 6m6-6l-6-6" />
		</svg>
	</button>

	

	<button class="btn btn-round btn-badge" disabled={$keyboardDisabled || !hintsAvailable} on:click={handleHint} title="Hints ({$hints})">
		<svg class="icon-outline" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
			<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.663 17h4.673M12 3v1m6.364 1.636l-.707.707M21 12h-1M4 12H3m3.343-5.657l-.707-.707m2.828 9.9a5 5 0 117.072 0l-.548.547A3.374 3.374 0 0014 18.469V19a2 2 0 11-4 0v-.531c0-.895-.356-1.754-.988-2.386l-.548-.547z" />
		</svg>

		{#if $settings.hintsLimited}
			<span class="badge" class:badge-primary={hintsAvailable}>{$hints}</span>
		{/if}
	</button>

	<!-- <button class="btn btn-round btn-badge" on:click={notes.toggle} title="Notes ({$notes ? 'ON' : 'OFF'})">
		<svg class="icon-outline" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
			<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.232 5.232l3.536 3.536m-2.036-5.036a2.5 2.5 0 113.536 3.536L6.5 21.036H3v-3.572L16.732 3.732z" />
		</svg>

		<span class="badge tracking-tighter" class:badge-primary={$notes}>{$notes ? 'ON' : 'OFF'}</span>
	</button> -->

</div>


<style>
	.action-buttons {
		@apply flex flex-wrap justify-evenly self-end;
	}

	.btn-badge {
		@apply relative;
	}

	.badge {
		min-height: 20px;
		min-width:  20px;
		@apply p-1 rounded-full leading-none text-center text-xs text-white bg-gray-600 inline-block absolute top-0 left-0;
	}

	.badge-primary {
		@apply bg-primary;
	}
</style>