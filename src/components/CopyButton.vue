<template>
<div
    :class="containerClasses"
    :aria-label="hint"
    aria-role="button"
    tabindex="0"
    @click.stop="handleClick"
    @keydown.space.enter="handleClick"
>
    <q-tooltip
        :offset="[0,0]"
        anchor="center end"
        self="center left"
    >
        {{ hint }}
    </q-tooltip>

    {{ accompanyingText }}

    <q-icon :class="iconClasses" />
</div>
</template>

<script>
import { copyToClipboard } from 'quasar'

const icons = {
    copy: 'far fa-copy',
    success: 'fas fa-check',
};

export default {
    name: 'CopyButton',
    props: {
        text: {
            type: String,
            required: true,
        },
        description: {
            type: String,
            default: '',
        },
        accompanyingText: {
            type: String,
            default: '',
        },
    },
    data: () => ({
        iconClass: icons.copy,
        hint: '',
    }),
    computed: {
        containerClasses() {
            const extraClass = this.accompanyingText ? 'c-copy-button--block' : '';
            return `c-copy-button ${extraClass}`;
        },
        iconClasses() {
            return `${this.iconClass} q-pl-xs`;
        },
        defaultHint() {
            return `Copy ${this.description} to clipboard`;
        },
    },
    created() {
        this.hint = this.defaultHint;
    },
    methods: {
        handleClick() {
            copyToClipboard(this.text).then(() => {
                this.iconClass = icons.success;
                this.hint = 'Copied';
                setTimeout(() => {
                    this.iconClass = icons.copy;
                    this.hint = this.defaultHint;
                }, 1500);
            }).catch((err) => {
                console.error(`Failed to copy to clipboard: ${err}`);
            })
        },
    },
}
</script>

<style lang="scss">
.c-copy-button {
    display: inline-flex;
    justify-content: center;
    margin-top: -2px;
    align-items: center;
    cursor: pointer;

    &--block {
        display: flex;
        width: max-content;
    }
}
</style>
