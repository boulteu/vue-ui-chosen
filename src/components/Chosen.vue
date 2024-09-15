<script setup>
    import { ref, computed, watch, watchEffect, nextTick } from 'vue';
    import List from './List.vue';

    const vClickOutside = {
        mounted: (el, binding) => {
            const clickHandler = (event) => {
                if (!el.contains(event.target)) {
                    if (typeof binding.value === 'function') {
                        binding.value(event);
                    }
                }
            };

            document.addEventListener('click', clickHandler);
            el.__clickOutsideHandler__ = clickHandler;
        },
        unmounted: (el) => {
            document.removeEventListener('click', el.__clickOutsideHandler__);
            delete el.__clickOutsideHandler__;
        }
    };

    const props = defineProps({
        values: {
            type: Object,
            required: true
        },
        multiple: {
            type: Boolean,
            default: false
        },
        loading: {
            type: Boolean,
            default: false
        },
        onScrollToListBottom: {
            type: Function,
            required: false
        },
        onSearch: {
            type: Function,
            required: false
        }
    });

    const divRef = ref(null);
    const selectRef = ref(null);
    const buttonsRef = ref(null);
    const inputRef = ref(null);
    const listRef = ref(null);

    const isOpen = ref(false);
    const setIsOpen = (value) => {
        isOpen.value = value;
    };

    const selectedValue = ref({});
    const setSelectedValue = value => {
        selectedValue.value = value;
    };

    const search = ref('');
    const setSearch = value => {
        search.value = value;
    };

    const pointer = ref(-1);
    const setPointer = value => {
        pointer.value = value;
    };

    const opened = ref({});

    const updateOpened = (key, value) => {
        opened.value[key] = value;
    };

    const selected = computed(() => {
        const arr = Object.keys(selectedValue.value);
        return props.multiple ? arr : arr[0];
    });

    const debounce = (func, delay) => {
        let timeoutId;
        return (...args) => {
            if (timeoutId) {
                clearTimeout(timeoutId);
            }
            timeoutId = setTimeout(() => {
                func(...args);
            }, delay);
        };
    };

    const flattenObject = (obj) => {
        const flattened = {};

        const flatten = (item) => {
            for (let key in item) {
                if (typeof item[key] === 'object' && item[key] !== null) {
                    flatten(item[key]);
                } else {
                    flattened[key] = item[key];
                }
            }
        };

        flatten(obj);
        return flattened;
    };

    const filterBySearch = (values) => {
        let results = {};

        for (let key in values) {
            const obj = values[key];

            if (typeof obj === 'object' && obj !== null) {
                const nestedResults = filterBySearch(obj);
                if (Object.keys(nestedResults).length > 0) {
                    results[key] = nestedResults;
                }
            } else if (obj && typeof obj === 'string' && obj.toLowerCase().includes(search.value.toLowerCase())) {
                results[key] = obj;
            }
        }

        return results;
    }

    const actionOnScroll = typeof props.onScrollToListBottom === 'function';
    const customSearch = typeof props.onSearch === 'function';

    const debouncedSearch = ref(debounce((value) => {
        if (customSearch) {
            props.onSearch(value);
        }
    }, 300));

    let filteredValues = {};
    let flattenFilteredValues = {};

    watchEffect(() => {
        filteredValues = customSearch ? props.values : filterBySearch(props.values);
        flattenFilteredValues = flattenObject(filteredValues);
    });

    const pointedValue = computed(() => {
        return Object.keys(flattenFilteredValues)[pointer.value];
    });

    const pointValue = (num) => {
        if (Object.keys(flattenFilteredValues)[pointer.value + num] !== undefined) {
            setPointer(pointer.value + num);
        }
    };

    const selectValue = (value) => {
        if (props.multiple ? !selected.value.includes(value) : selected.value !== value) {
            setSelectedValue({
                ...(props.multiple ? selectedValue.value : {}),
                ...Object.fromEntries(Object.entries(flattenFilteredValues).filter(([key, _]) => key === value))
            });
            setIsOpen(false);
        }
    };

    const unselectValue = (value) => {
        if (props.multiple ? selected.value.includes(value) : selected.value === value) {
            setSelectedValue(Object.fromEntries(Object.entries(selectedValue.value).filter(([key, _]) => key !== value)));
        }
    };

    const removeLastSelected = () => {
        if (!search.value) {
            const keys = Object.keys(selectedValue.value);
            if (keys.length) {
                unselectValue(keys[keys.length - 1]);
            }
        }
    };

    const handleKeyDown = (e) => {
        switch (e.key) {
            case 'ArrowUp':
                pointValue(-1);
                break
            case 'ArrowDown':
                if (!isOpen.value) {
                    setIsOpen(true)
                } else {
                    pointValue(1);
                }
                break
            case 'Enter':
                selectValue(pointedValue.value);
                break
            case 'Backspace':
                removeLastSelected();
                break
        }
    }

    const handleBulk = (event, value) => {
        event.stopPropagation();
        setSelectedValue(value);
        setIsOpen(false);
    }

    watch(isOpen, async () => {
        if (isOpen.value) {
            await nextTick();

            if (inputRef.value) {
                inputRef.value.focus();
            }

            if (selectRef.value && buttonsRef.value) {
                const handleScroll = () => {
                    buttonsRef.value.style.top = `${selectRef.value.scrollTop + 2}px`;
                };

                selectRef.value.addEventListener('scroll', handleScroll);

                return () => {
                    if (selectRef.value) {
                        selectRef.value.removeEventListener('scroll', handleScroll);
                    }
                    if (buttonsRef.value) {
                        buttonsRef.value.style.top = '';
                    }
                };
            }
        } else {
            setSearch('');
        }
    });

    watch(search, () => {
        if (customSearch) {
            debouncedSearch(search);
        }
    });

    watch([isOpen, search], () => {
        setPointer(-1);

        if (actionOnScroll.value && listRef.value) {
            const handleScroll = () => {
                const alreadyScrolledHeight = listRef.value.clientHeight + (listRef.value.pageYOffset || listRef.value.scrollTop)
                if (alreadyScrolledHeight >= listRef.value.scrollHeight) {
                    props.onScrollToListBottom(search.value)
                }
            };

            listRef.value.addEventListener('scroll', handleScroll);

            return () => {
                if (listRef.value) {
                    listRef.value.removeEventListener('scroll', handleScroll);
                }
            };
        }
    });
</script>

<template>
    <div class="relative" v-click-outside="() => { setIsOpen(false) }" ref="divRef">
        <ul class="relative bg-white border border-slate-400 rounded py-0.5 pl-1 pr-5 min-h-[2.25rem] max-h-28 overflow-y-auto cursor-pointer" :class="{ 'border-b-0 rounded-b-none': isOpen }" @click="setIsOpen(true)" :ref="selectRef" v-if="props.multiple">
            <template v-for="key in selected" :key="key">
                <li class="relative float-left border border-slate-400 rounded m-1 ml-0 p-1 pr-5 leading-3 text-gray-700 selectChoice">
                    <span>
                        {{ selectedValue[key] }}
                    </span>
                    <a class="absolute top-0.5 right-0.5 w-4 h-4 cursor-pointer" @click="unselectValue(key)">
                        <svg
                            fill="none"
                            viewBox="0 0 24.00 24.00"
                            xmlns="http://www.w3.org/2000/svg"
                            class="stroke-2 stroke-slate-400 hover:stroke-slate-500"
                        >
                            <g>
                                <path
                                    d="M10.9393 12L6.9696 15.9697L8.03026 17.0304L12 13.0607L15.9697 17.0304L17.0304 15.9697L13.0607 12L17.0303 8.03039L15.9696 6.96973L12 10.9393L8.03038 6.96973L6.96972 8.03039L10.9393 12Z"
                                ></path>
                            </g>
                        </svg>
                    </a>
                </li>
            </template>
            <li class="float-left">
                <input
                    type="text"
                    autoComplete="off"
                    class="p-0 h-6 bg-transparent border-none mt-0.5 focus:outline-none focus:border-transparent focus:ring-0" :class="selected.length ? 'w-6' : 'w-full'"
                    v-model="search"
                    @keydown="handleKeyDown"
                    ref="inputRef"
                />
            </li>
            <li class="clear-left w-4 h-4 absolute right-0.5" ref="buttonsRef">
                <div v-if="isOpen">
                    <a @click="(e) => handleBulk(e, flattenFilteredValues)">
                        <svg
                            fill="none"
                            viewBox="0 0 24.00 24.00"
                            xmlns="http://www.w3.org/2000/svg"
                            class="stroke-2 stroke-slate-400 hover:stroke-slate-500"
                        >
                            <g>
                                <path
                                    d="M6 12H18M12 6V18"
                                ></path>
                            </g>
                        </svg>
                    </a>
                    <a @click="(e) => handleBulk(e, {})">
                        <svg
                            fill="none"
                            viewBox="0 0 24.00 24.00"
                            xmlns="http://www.w3.org/2000/svg"
                            class="stroke-2 stroke-slate-400 hover:stroke-slate-500"
                        >
                            <g>
                                <path
                                    d="M6 12L18 12"
                                ></path>
                            </g>
                        </svg>
                    </a>
                </div>
            </li>
        </ul>
        <a
            class="relative block h-9 pl-2 border border-slate-400 rounded leading-6 cursor-pointer" :class="isOpen ? 'selectOpen border-b-0 rounded-b-none' : 'selectSingle'"
            @click="setIsOpen(!isOpen)"
            v-if="!props.multiple"
        >
            <span class="block truncate mt-1 mr-6">
                {{ selectedValue[selected] }}
            </span>
            <svg
                viewBox="0 0 24 24"
                xmlns="http://www.w3.org/2000/svg"
                class="absolute top-1/3 right-2 w-2 h-2 fill-slate-400"
                :transform="isOpen ? 'rotate(180)' : ''"
            >
                <g>
                    <path
                        d="M11.178 19.569a.998.998 0 0 0 1.644 0l9-13A.999.999 0 0 0 21 5H3a1.002 1.002 0 0 0-.822 1.569l9 13z"
                    ></path>
                </g>
            </svg>
        </a>
        <div class="border border-t-0 border-slate-400" :class="!props.multiple ? 'rounded-b' : 'rounded-b'" v-if="isOpen">
            <div class="relative p-1" v-if="!props.multiple">
                <input
                    type="text"
                    autoComplete="off"
                    class="bg-white border border-slate-400 w-full h-8 pl-1 pr-6 focus:outline-none focus:border-slate-400 focus:ring-0"
                    v-model="search"
                    @keydown="handleKeyDown"
                    ref="inputRef"
                />
                <svg
                    fill="none"
                    viewBox="0 0 24 24"
                    xmlns="http://www.w3.org/2000/svg"
                    class="absolute top-1/4 right-2 w-5 h-5 stroke-2 stroke-slate-400"
                >
                    <g>
                        <path
                            d="M14.9536 14.9458L21 21M17 10C17 13.866 13.866 17 10 17C6.13401 17 3 13.866 3 10C3 6.13401 6.13401 3 10 3C13.866 3 17 6.13401 17 10Z"
                        ></path>
                    </g>
                </svg>
            </div>
            <ul class="overflow-y-auto" :class="props.multiple ? 'max-h-64' : 'max-h-52'" ref="listRef">
                <List
                    v-for="(value, key) in filteredValues"
                    :title="key"
                    :value="value"
                    :selected="selected"
                    :opened="opened[key]"
                    :pointed-value="pointedValue"
                    :select-callback="selectValue"
                    :open-all="!!(search || pointedValue)"
                    :update-opened="updateOpened"
                />
                <li class="p-1 text-slate-400 hover:cursor-not-allowed" v-if="!Object.keys(props.values).length">
                    No results match
                </li>
            </ul>
        </div>
        <div v-if="props.loading">
            <div class="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 z-10 flex items-center">
                <button type="button" class="inline-flex items-center font-semibold leading-6 text-sm text-white rounded-md cursor-not-allowed" disabled>
                    <svg class="animate-spin h-5 w-5 text-slate-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                    </svg>
                </button>
            </div>
            <div class="absolute inset-0 z-40 bg-gray-100 bg-opacity-50"></div>
        </div>
    </div>
</template>

<style>
    .selectOpen {
        background: linear-gradient(#eee 20%, #fff 80%);
    }
    .selectSingle {
        background: linear-gradient(#fff 20%, #f6f6f6 50%, #eee 52%, #f4f4f4 100%);
    }
    .selectChoice {
        background: linear-gradient(#f4f4f4 20%, #f0f0f0 50%, #e8e8e8 52%, #eee 100%);
    }
</style>
