<script setup>
    const props = defineProps({
        value: [Object, Array, String],
        title: String,
        selected: [Object, Array, String],
        opened: Boolean,
        pointedValue: String,
        selectCallback: Function,
        openAll: Boolean,
        updateOpened: Function
    });

    const handleClick = (e) => {
        e.stopPropagation();
        props.selectCallback(props.title);
    };

    const toggleOpen = (e) => {
        e.stopPropagation();
        props.updateOpened(props.title, !props.opened);
    };

    const isObject = (val) => typeof val === 'object' && val !== null;

    const selectedResult = 'selectedResult text-white';
    const selectResult = 'selectResult hover:text-white hover:cursor-pointer';
    const isSelected = (value) => {
        return Array.isArray(props.selected) ? props.selected.includes(value) : props.selected === value;
    };
</script>

<template>
    <li
        :class="{
            'p-1': true,
            'text-slate-400 hover:cursor-not-allowed': isSelected(title),
            [selectedResult]: pointedValue !== undefined && title === pointedValue,
            [selectResult]: !isObject(value) && !isSelected(title)
        }"
        @click.stop="handleClick"
    >
        <div v-if="isObject(value)"
             :class="['relative font-bold p-1 hover:cursor-pointer']"
             @click.stop="toggleOpen"
        >
            {{ title }}
            <svg
                viewBox="0 0 24 24"
                xmlns="http://www.w3.org/2000/svg"
                class="absolute top-1/3 right-2 w-2 h-2 fill-slate-400"
                :transform="(opened || openAll) ? 'rotate(180)' : ''"
            >
                <g>
                    <path
                        d="M11.178 19.569a.998.998 0 0 0 1.644 0l9-13A.999.999 0 0 0 21 5H3a1.002 1.002 0 0 0-.822 1.569l9 13z"
                    ></path>
                </g>
            </svg>
        </div>
        <ul v-if="isObject(value)" v-show="(opened || openAll)">
            <List
                v-for="(item, key) in value"
                :title="key"
                :value="item"
                :selected="selected"
                :opened="opened"
                :pointed-value="pointedValue"
                :select-callback="selectCallback"
                :open-all="openAll"
                :update-opened="updateOpened"
            />
        </ul>
        <span v-else>{{ value }}</span>
    </li>
</template>

<style>
    .selectedResult, .selectResult:hover {
        background: linear-gradient(#3875d7 20%, #2a62bc 90%);
    }
</style>
