<script>
	import Timer from './Timer.svelte';
	import Actions from './Actions.svelte';
	import StrategyDescription from './StrategyDescription.svelte';
</script>

<div class="action-bar space-y-3 xs:space-y-0">
	<Timer />

	<div class="action-section">
		<Actions />
		<StrategyDescription />
	</div>
</div>

<style>
	.action-bar {
		@apply flex flex-col flex-wrap justify-between pb-5;
	}

	.action-section {
		@apply flex flex-col;
	}

	@screen xs {
		.action-bar {
			@apply flex-row;
		}
	}
</style>