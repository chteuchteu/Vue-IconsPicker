<template>
    <div
        class="app"
        :class="{
            ['accent-color--'+accentColor]: true,
            'random-icon-colors': randomColors
        }"
    >
        <header class="header">
            <div class="title">
                <h1><i :class="config.icons.main"></i> {{ config.name }}</h1>
                <small class="version">v {{ version }}</small>
            </div>
            <div class="filter-wrap">
                <input
                    class="filter" type="text"
                    placeholder="Search..." autofocus
                    v-model="filter"
                />
                <i
                    :class="{
                        [config.icons.close]: true,
                        hidden: filter.length === 0
                    }"
                    @click="filter = ''"
                ></i>
            </div>
        </header>

        <div class="content">
            <div v-if="loading" class="loading">
                <i :class="config.icons.loading"></i> Loading&hellip;
            </div>

            <div
                class="icons"
                ref="icons"
                @click="setActiveIcon(null)"
            >
                <i
                    v-for="icon in filteredIcons"
                    :class="{
                        [config.classPrefix]: true,
                        [config.classPrefix+'-'+icon.name]: true,
                        active: isIconActive && activeIcon === icon
                    }"
                    @click.stop="setActiveIcon(icon)"
                    :title="icon.name"
                ></i>
            </div>

            <transition name="properties">
                <div class="icon-properties" v-show="isIconActive">
                    <i
                        class="icon-icon"
                        :class="activeIcon && config.classPrefix+' '+config.classPrefix+'-'+activeIcon.name"
                        @click="changeAccentColor"
                    ></i>

                    <div class="properties">
                        <span class="icon-name">{{ activeIcon && activeIcon.name }}</span>
                        <input
                            class="icon-class"
                            placeholder="Class"
                            :value="config.classPrefix + ' ' + config.classPrefix + '-' + (activeIcon && activeIcon.name)"
                            disabled
                        />
                        <p class="icon-more">
                            <span class="icon-codepoint">{{ activeIcon && activeIcon.codepoint }}</span>
                            <span
                                class="icon-version-wrap"
                                v-show="activeIcon && activeIcon.version"
                            > &bullet; v<span class="icon-version">{{ activeIcon && activeIcon.version }}</span></span>
                            <span
                                class="icon-author-wrap"
                                v-show="activeIcon && activeIcon.author"
                            > &bullet; by <span class="icon-author">{{ activeIcon && activeIcon.author }}</span></span>
                        </p>
                    </div>
                </div>
            </transition>
        </div>

        <footer>
            <div class="left">
                <a
                    class="btn-sm-round"
                    :href="isIconActive ? config.openIconUrl.replace('{icon}', activeIcon.name) : config.openUrl"
                    target="_blank"
                    :title="isIconActive ? config.openIconText.replace('{icon}', activeIcon.name) : config.openText"
                >
                    <i :class="config.icons.openExternal"></i>
                </a>
                <a
                    class="btn-sm-round"
                    href="#"
                    @click.prevent="setActiveIcon(getRandomIcon(), true)"
                    title="Random icon"
                >
                    <i :class="config.icons.random"></i>
                </a>
                <a
                    class="btn-sm-round"
                    href="#"
                    @click.prevent="randomColors = !randomColors"
                    title="Random colors"
                >
                    <i :class="config.icons.randomColors"></i>
                </a>
                <a
                    v-if="config.canCopySvg"
                    class="btn-sm-round"
                    href="#"
                    v-show="isIconActive"
                    @click.prevent="copySvg"
                    title="Copy SVG"
                >
                    <i :class="config.icons.svg"></i>
                </a>
            </div>
            <div class="right">
                <a
                    class="btn-sm-round"
                    href="https://www.s-quent.in"
                    target="_blank"
                    title="Made with <3 by Quentin S."
                >
                    <i :class="config.icons.madeBy"></i>
                </a>
                <a
                    class="btn-sm-round"
                    :href="config.repoUrl"
                    target="_blank"
                    title="GitHub"
                >
                    <i :class="config.icons.gitHub"></i>
                </a>
            </div>
            <div class="clearfix"></div>
            <input ref="input-copy" style="display: none" />
        </footer>
    </div>
</template>

<script>
    import { request } from './lib/Request';
    import { randomInt } from './lib/Math';
    import { computeOffset, outerHeight } from './lib/jQuery';

    const ACCENT_COLOR_KEY = 'color-accent';
    const RANDOM_COLORS_KEY = 'random-colors';
    const COLORS = [
        'red', 'pink', 'purple', 'deep-purple', 'indigo', 'blue', 'light-blue', 'cyan', 'teal', 'green',
        'light-green', 'lime', 'amber', 'orange', 'deep-orange', 'brown', 'grey', 'blue-grey'
    ];

    const filterReplaceRegex = new RegExp('-', 'g');

    const getResourceUrl = (filename) => typeof(chrome) !== 'undefined' && chrome.extension !== undefined
        ? chrome.extension.getURL('dist/data/' + filename)
        : '../data/' + filename; // <- when debugging extension directly from index.html

    export default {
        name: 'icons-picker',
        props: {
            config: Object,
        },
        data: () => ({
            loading: true,
            filter: '',
            icons: [],
            version: null,
            svg: [],

            accentColor: null,
            randomColors: false,
            activeIcon: null,
            isIconActive: false,
        }),
        computed: {
            filteredIcons() {
                const filterVal = this.filter.replace(filterReplaceRegex, ' ');

                return this.icons.filter((icon) => {
                    return icon.searchable.indexOf(filterVal) !== -1;
                });
            }
        },
        mounted() {
            // Load accent color settings
            this.accentColor = localStorage.getItem(ACCENT_COLOR_KEY) || 'orange';
            this.randomColors = JSON.parse(localStorage.getItem(RANDOM_COLORS_KEY)) || false;

            // Load icons
            request(getResourceUrl('icons.min.json'))
                .then(JSON.parse)
                .then((response) => {
                    this.loading = false;
                    this.icons = response.icons;
                    this.version = response.version;
                });

            this.loading = false;

            // Load svg
            if (this.config.canCopySvg) {
                request(getResourceUrl('icons-svg.min.json'))
                    .then(JSON.parse)
                    .then((response) => {
                        this.svg = response;
                    });
            }
        },
        methods: {
            setActiveIcon(icon, ensureVisible=false) {
                const hasActiveIcon = this.isIconActive;
                this.isIconActive = icon !== null;

                if (icon !== null) {
                    this.activeIcon = icon;

                    if (ensureVisible) {
                        // Wait for the UI to be updated for calculations
                        const iconElem = document.querySelector('.'+this.config.classPrefix+'-'+icon.name),
                            iconsList = this.$refs['icons'];

                        const offset = computeOffset(iconElem).top - computeOffset(iconsList).top,
                            iconElemHeight = outerHeight(iconElem, true);

                        // Take properties pannel height into account when it appears
                        // (it's not shown yet)
                        let iconsListHeight = iconsList.clientHeight;
                        if (!hasActiveIcon) {
                            iconsListHeight -= 80;
                        }

                        let scrollTop = iconsList.scrollTop;
                        if (offset - 5 < 0) {
                            scrollTop += offset -5;
                        }
                        else if (offset + iconElemHeight > iconsListHeight) {
                            scrollTop += offset + iconElemHeight - iconsListHeight;
                        }

                        iconsList.scrollTop = scrollTop;
                    }
                }
            },
            getRandomIcon() {
                return this.icons[randomInt(0, this.icons.length-1)];
            },
            changeAccentColor() {
                const i = COLORS.indexOf(this.accentColor)+1;
                this.accentColor = COLORS[i > COLORS.length-1 ? 0 : i];
            },
            copySvg() {
                if (!this.config.canCopySvg) {
                    throw new Error('You\'re not supposed to do this!');
                }
                if (this.activeIcon === null) {
                    return;
                }

                const name = this.activeIcon.name,
                    svg = this.svg[name],
                    input = this.$refs['input-copy'];

                input.value = svg;
                input.select();

                document.oncopy = function(event) {
                    event.clipboardData.setData('text/plain', svg);
                    event.preventDefault();
                };
                document.execCommand("Copy", false, null);
            }
        },
        watch: {
            accentColor() {
                localStorage.setItem(ACCENT_COLOR_KEY, this.accentColor);
            },
            randomColors() {
                localStorage.setItem(RANDOM_COLORS_KEY, this.randomColors.toString());
            }
        }
    };
</script>
