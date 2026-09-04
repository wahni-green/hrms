<template>
	<Popover placement="bottom-end" v-model:show="isOpen" :hide-on-blur="false">
		<template #target="{ togglePopover }">
			<div ref="triggerRef">
				<Button variant="subtle" icon-left="filter" @click="togglePopover">
					Filters
					<span
						v-if="activeCount"
						class="ml-1.5 flex h-4 w-4 items-center justify-center rounded-full bg-gray-900 text-xs text-white"
					>
						{{ activeCount }}
					</span>
				</Button>
			</div>
		</template>
		<template #body-main>
			<div class="w-80 space-y-2 p-3">
				<div class="flex items-center justify-between">
					<span class="text-sm font-medium text-gray-600">Select Filters</span>
					<Button variant="ghost" @click="clearAll" v-if="rows.length">Clear Filters</Button>
				</div>

				<p v-if="!rows.length" class="text-sm text-gray-500">No filters added.</p>

				<div v-for="row in rows" :key="row.id" class="flex items-center gap-2">
					<div class="w-32">
						<FormControl
							type="autocomplete"
							placeholder="Field"
							:options="fieldOptions(row)"
							:model-value="row.field ? { label: row.field.label, value: row.field.fieldname } : null"
							@update:model-value="(opt) => onFieldChange(row, opt)"
						/>
					</div>

					<FormControl
						v-if="row.field?.fieldtype === 'Select'"
						type="select"
						class="w-32"
						placeholder="Value"
						:options="selectOptions(row.field)"
						v-model="row.value"
					/>
					<FormControl
						v-else-if="row.field?.fieldtype === 'Link'"
						type="autocomplete"
						class="w-32"
						placeholder="Value"
						:options="row.linkOptions"
						:model-value="row.value ? { label: row.value, value: row.value } : null"
						@update:model-value="(opt) => (row.value = opt?.value || null)"
						:disabled="!row.field"
					/>
					<FormControl
						v-else-if="row.field?.fieldtype === 'Check'"
						type="select"
						class="w-32"
						placeholder="Value"
						:options="['Yes', 'No']"
						v-model="row.value"
					/>
					<FormControl
						v-else
						:type="textInputType(row.field)"
						class="w-32"
						placeholder="Value"
						v-model="row.value"
						:disabled="!row.field"
					/>

					<Button icon="x" variant="ghost" @click="removeRow(row)" />
				</div>

				<Button variant="subtle" icon-left="plus" @click="addRow">Add Filter</Button>
			</div>
		</template>
	</Popover>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from "vue";
import { Button, FormControl, Popover, createListResource, createResource } from "frappe-ui";

export type EmployeeField = {
	fieldname: string;
	label: string;
	fieldtype: string;
	options: string | null;
};

type FilterRow = {
	id: number;
	field: EmployeeField | null;
	value: string | null;
	linkOptions: string[];
};

const props = defineProps<{
	// fields already handled by other filter controls, hidden from the picker
	excludeFields?: string[];
}>();

const emit = defineEmits<{
	(e: "update:filters", filters: Record<string, string>): void;
}>();

let nextId = 0;
const rows = reactive<FilterRow[]>([]);
const isOpen = ref(false);
const triggerRef = ref<HTMLElement>();

const activeCount = computed(() => rows.filter((row) => row.field && row.value).length);

// frappe-ui's Popover has its own outside-click detector, but the Field
// autocomplete's dropdown is *also* a Popover, teleported to the same shared
// root as a sibling rather than a DOM descendant of this one - so the
// built-in detector can't tell "clicked a nested dropdown" apart from
// "clicked outside" and closes this panel the moment a field is picked.
// hideOnBlur=false above turns that off; this replaces it with a check that
// treats any click landing inside the shared popper root (i.e. any open
// dropdown, including nested ones) as "still inside".
const onDocumentMousedown = (event: MouseEvent) => {
	const target = event.target as Node;
	if (triggerRef.value?.contains(target)) return;
	const popperRoot = document.getElementById("frappeui-popper-root");
	if (popperRoot?.contains(target)) return;
	isOpen.value = false;
};

onMounted(() => document.addEventListener("mousedown", onDocumentMousedown));
onBeforeUnmount(() => document.removeEventListener("mousedown", onDocumentMousedown));

const availableFields = createResource({
	url: "hrms.api.roster.get_employee_filter_fields",
	auto: true,
	initialData: [] as EmployeeField[],
});

const selectableFields = computed(() =>
	(availableFields.data as EmployeeField[]).filter(
		(f) => !(props.excludeFields || []).includes(f.fieldname),
	),
);

const fieldOptions = (row: FilterRow) =>
	selectableFields.value
		.filter((f) => f.fieldname === row.field?.fieldname || !rows.some((r) => r.field?.fieldname === f.fieldname))
		.map((f) => ({ label: f.label, value: f.fieldname }));

const selectOptions = (field: EmployeeField) => (field.options || "").split("\n").filter(Boolean);

const textInputType = (field: EmployeeField | null) => {
	if (!field) return "text";
	if (["Int", "Float", "Currency", "Percent", "Duration"].includes(field.fieldtype)) return "number";
	if (field.fieldtype === "Date") return "date";
	if (field.fieldtype === "Datetime") return "datetime-local";
	return "text";
};

const addRow = () => {
	rows.push({ id: nextId++, field: null, value: null, linkOptions: [] });
};

const removeRow = (row: FilterRow) => {
	const index = rows.indexOf(row);
	if (index !== -1) rows.splice(index, 1);
};

const clearAll = () => {
	rows.splice(0, rows.length);
};

defineExpose({ clearAll });

const onFieldChange = (row: FilterRow, opt: { label: string; value: string } | null) => {
	const field = opt ? selectableFields.value.find((f) => f.fieldname === opt.value) || null : null;
	row.field = field;
	row.value = null;
	row.linkOptions = [];

	if (field?.fieldtype === "Link" && field.options) {
		createListResource({
			doctype: field.options,
			fields: ["name"],
			pageLength: 100,
			auto: true,
			onSuccess: (data: { name: string }[]) => {
				row.linkOptions = data.map((item) => item.name);
			},
		});
	}
};

watch(
	rows,
	(val) => {
		const filters: Record<string, string> = {};
		val.forEach((row) => {
			if (row.field && row.value) filters[row.field.fieldname] = row.value;
		});
		emit("update:filters", filters);
	},
	{ deep: true },
);
</script>
