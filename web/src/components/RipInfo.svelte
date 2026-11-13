<script lang="ts">
	import IconCdCreateExchange from '~icons/carbon/cd-create-exchange';
	import IconRadar from '~icons/carbon/radar';
	import IconIbmWatsonLanguageTranslator from '~icons/carbon/ibm-watson-language-translator';
	import IconWorkspace from '~icons/carbon/workspace';
	import IconShowDataCards from '~icons/carbon/show-data-cards';
	import IconDocumentWordProcessorReference from '~icons/carbon/document-word-processor-reference';
	import IconCdArchive from '~icons/carbon/cd-archive';
	import InfoSegment from './frags/InfoSegment.svelte';
	import IconCheckmarkFilled from '~icons/carbon/checkmark-filled';
	import IconCloseFilled from '~icons/carbon/close-filled';
	import Card from './frags/Card.svelte';
	import type { ParsedLog } from '#lib/types/ParsedLog.js';
	import type { Evaluation } from '#lib/types/Evaluation.js';
	import type { EvaluationUnitField } from '#lib/types/EvaluationUnitField.js';

	interface Props {
		parsedLog: ParsedLog;
		evaluation: Evaluation;
	}

	let { parsedLog, evaluation }: Props = $props();

	const evMap: Record<EvaluationUnitField, number> = $derived.by(() => {
		const map: Record<EvaluationUnitField, number> = {
			Encoding: 0,
			RipperVersion: 0,
			Drive: 0,
			Ripper: 0,
			Offset: 0,
			Cache: 0,
			TestAndCopy: 0,
			Encoder: 0,
			Checksum: 0,
			MediaType: 0,
			ReadMode: 0,
			MaxRetryCount: 0,
			AccurateStream: 0,
			C2: 0,
			SilentSamples: 0,
			NullSamples: 0,
			Gap: 0,
			Tag: 0,
			Gain: 0,
			RangeSplit: 0,
			Samples: 0,
			SilentBlocks: 0,
			Normalization: 0,
			Filename: 0,
			ReadError: 0,
			SkipError: 0,
			JitterGenericError: 0,
			JitterEdgeError: 0,
			JitterAtomError: 0,
			DriftError: 0,
			DroppedError: 0,
			DuplicatedError: 0,
			InconsistentErrorSectors: 0,
			DamagedSector: 0,
			Abort: 0
		};

		evaluation.evaluation_units.forEach((unit) => {
			if (unit.data.scope === 'Release') {
				const { field } = unit.data;
				map[field] = map[field] + 1;
			}
		});

		return map;
	});

	const chipClass = 'preset-tonal-primary rounded-md text-xs px-2 py-1 font-semibold';

	// Mirrors `OpsEvaluator::is_image_rip()` in the Rust evaluator: EAC writes a
	// single `Range status and errors` block when the whole disc is extracted into
	// one file, and such a log carries no gap handling line at all.
	const isEacImageRip = $derived.by(() => {
		if (parsedLog.ripper !== 'Exact Audio Copy') return false;

		const tracks = parsedLog.tracks;
		if (tracks.length !== 1 || !tracks[0].is_range) return false;

		const entries = parsedLog.toc.raw.entries;
		if (entries.length === 0) return false;

		const first = entries[0];
		const last = entries[entries.length - 1];

		return first.start_sector === 0 && parsedLog.toc.raw.lead_out === last.end_sector + 1;
	});

	// For an image rip the gap handling genuinely cannot be verified, but no
	// evaluation unit is produced (the rule is exempted, so the score stays at
	// 100). Mark the field anyway so "Unknown" is not shown as a green tick.
	const gapUnverifiable = $derived(isEacImageRip && parsedLog.gap_handling === 'Unknown');
</script>

<Card header="Rip Info">
	{#snippet tooltip()}
		<div class="flex flex-col gap-2">
			<div class="flex flex-col gap-2 text-xs">
				<p class="max-w-64 text-balance">
					The ripper settings should be adjusted to ensure that all the values match the
					recommendation. Note that some <strong>rippers</strong>, albeit being
					<strong>functionally fine</strong>, might be marked as <em>not recommended</em> due to the lack
					of private tracker support.
				</p>
				<div class="flex gap-2 items-center">
					<IconCheckmarkFilled class="text-success-700 visible dark:text-success-400 icon-sm" />
					<span>Matches recommended value</span>
				</div>
				<div class="flex gap-2 items-center">
					<IconCloseFilled class="text-error-700 dark:text-error-400 icon-sm" />
					<span>Not the recommended value</span>
				</div>
			</div>
		</div>
	{/snippet}
	{#if isEacImageRip}
		<p class="text-xs mb-3 text-warning-700 dark:text-warning-400">
			Image rip (single file) — EAC does not log gap handling for these extractions.
		</p>
	{/if}
	<div class="flex flex-col gap-4">
		<InfoSegment
			icon={IconCdCreateExchange}
			header="Ripper"
			value={parsedLog.ripper}
			valueOk={evMap['Ripper'] + evMap['RipperVersion']}
		>
			{#snippet extra()}
				<div class={chipClass}>
					{parsedLog.ripper_version === 'Unknown' ? '' : 'v'}{parsedLog.ripper_version}
				</div>
			{/snippet}
		</InfoSegment>
		<InfoSegment
			icon={IconDocumentWordProcessorReference}
			header="Read mode"
			value={parsedLog.read_mode}
			valueOk={evMap['ReadMode']}
		/>
		<!-- TODO: Merge this with drive -->
		<InfoSegment
			icon={IconRadar}
			header="Combined R/W offset"
			value={parsedLog.combined_rw_offset}
		/>
		<InfoSegment
			icon={IconIbmWatsonLanguageTranslator}
			header="Language"
			value={parsedLog.language}
		/>
		<InfoSegment
			icon={IconWorkspace}
			header="Gap handling"
			value={parsedLog.gap_handling}
			valueOk={evMap['Gap'] > 0 || gapUnverifiable ? 1 : 0}
		/>
		<InfoSegment
			icon={IconShowDataCards}
			header="Drive"
			value={parsedLog.drive}
			valueOk={evMap['Drive']}
		>
			{#snippet extra()}
				<div class={chipClass}>
					{parsedLog.read_offset && parsedLog.read_offset > 0 ? '+' : ''}{parsedLog.read_offset}
				</div>
			{/snippet}
		</InfoSegment>
		{#if parsedLog.media_type !== 'Unknown'}
			<InfoSegment icon={IconCdArchive} header="Media type" value={parsedLog.media_type} />
		{/if}
	</div>
</Card>
