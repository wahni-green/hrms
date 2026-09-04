<template>
	<div class="flex items-center justify-between">
		<!-- Month Change -->
		<div class="flex items-center bg-gray-50 rounded-md space-x-0.5">
			<Button icon="chevron-left" variant="ghost" @click="emit('addToMonth', -1)" />
			<span class="w-32 text-center font-medium text-base">
				{{ props.firstOfMonth.format("MMMM") }}, {{ firstOfMonth.format("YYYY") }}
			</span>
			<Button icon="chevron-right" variant="ghost" @click="emit('addToMonth', 1)" />
		</div>

		<!-- Filters -->
		<div class="flex flex-wrap items-center justify-end gap-2.5">
			<div v-for="[key, filter] of Object.entries(filters)" :key="key" class="filter-item w-40">
				<FormControl
					type="autocomplete"
					:placeholder="toTitleCase(key)"
					:options="filter.options"
					v-model="filter.model"
					:disabled="!filter.options.length"
				/>
			</div>

			<!-- Field-driven filters: pick any Employee field instead of hardcoding one control per field -->
			<DynamicEmployeeFilters
				ref="dynamicFiltersRef"
				:exclude-fields="pinnedFieldnames"
				@update:filters="onDynamicFiltersChange"
			/>

			<Button icon="x" @click="clearAllFilters" />
		</div>
	</div>
</template>

<style scoped>
/* frappe-ui's Autocomplete placeholder span lacks a truncate class, so long
   placeholders (e.g. "Employment Type") wrap to two lines and misalign the
   filter row's height. Force it to a single truncated line here instead. */
.filter-item :deep(button span) {
	white-space: nowrap;
	overflow: hidden;
	text-overflow: ellipsis;
}
</style>

<script setup lang="ts">
import { reactive, ref, watch } from "vue";
import { FormControl, createResource, createListResource } from "frappe-ui";
import { Dayjs } from "dayjs";

import { raiseToast } from "../utils";
import DynamicEmployeeFilters from "./DynamicEmployeeFilters.vue";

// Filters with special behaviour (company gates the whole table, department
// cascades from company) stay as dedicated pinned controls. Every other
// Employee field is filterable through DynamicEmployeeFilters, which reads
// the field list from the server instead of hardcoding it here.
type PinnedField = "company" | "department" | "branch" | "designation" | "shift_type" | "shift_location";

const pinnedFieldnames: PinnedField[] = [
	"company",
	"department",
	"branch",
	"designation",
	"shift_type",
	"shift_location",
];

type PinnedFilter = {
	doctype: string;
	options: string[];
	model?: { value: string } | null;
};

const props = defineProps<{
	firstOfMonth: Dayjs;
}>();

const emit = defineEmits<{
	(e: "addToMonth", change: number): void;
	(e: "updateFilters", newFilters: Record<string, string>): void;
}>();

const PINNED_DOCTYPES: Record<PinnedField, string> = {
	company: "Company",
	department: "Department",
	branch: "Branch",
	designation: "Designation",
	shift_type: "Shift Type",
	shift_location: "Shift Location",
};

const filters: Record<PinnedField, PinnedFilter> = reactive(
	Object.fromEntries(
		pinnedFieldnames.map((field) => [field, { doctype: PINNED_DOCTYPES[field], options: [], model: null }]),
	),
) as Record<PinnedField, PinnedFilter>;

let dynamicFilters: Record<string, string> = {};

watch(
	() => filters.company.model,
	(val) => {
		if (val?.value) getFilterOptions("department", { company: val.value });
		else {
			filters.department.model = null;
			filters.department.options = [];
		}
	},
);

watch(filters, (val) => {
	const pinnedFilters = Object.fromEntries(
		(Object.keys(val) as PinnedField[]).map((key) => [key, val[key].model?.value || ""]),
	);
	emit("updateFilters", { ...pinnedFilters, ...dynamicFilters });
});

const onDynamicFiltersChange = (newDynamicFilters: Record<string, string>) => {
	dynamicFilters = newDynamicFilters;
	const pinnedFilters = Object.fromEntries(
		(Object.keys(filters) as PinnedField[]).map((key) => [key, filters[key].model?.value || ""]),
	);
	emit("updateFilters", { ...pinnedFilters, ...dynamicFilters });
};

const dynamicFiltersRef = ref<InstanceType<typeof DynamicEmployeeFilters>>();

const clearAllFilters = () => {
	Object.values(filters).forEach((d) => (d.model = null));
	dynamicFiltersRef.value?.clearAll();
};

const toTitleCase = (str: string) =>
	str
		.split("_")
		.map((s) => s.charAt(0).toUpperCase() + s.slice(1))
		.join(" ");

// RESOURCES

const defaultCompany = createResource({
	url: "hrms.api.roster.get_default_company",
	auto: true,
	onSuccess: () => {
		pinnedFieldnames.forEach((field) => getFilterOptions(field));
	},
});

const getFilterOptions = (field: PinnedField, listFilters: { company?: string } = {}) => {
	createListResource({
		doctype: filters[field].doctype,
		fields: ["name"],
		filters: listFilters,
		pageLength: 100,
		auto: true,
		onSuccess: (data: { name: string }[]) => {
			const value = field === "company" ? defaultCompany.data : "";
			filters[field].model = { value };
			filters[field].options = data.map((item) => item.name);
		},
		onError(error: { messages: string[] }) {
			raiseToast("error", error.messages[0]);
		},
	});
};
</script>
