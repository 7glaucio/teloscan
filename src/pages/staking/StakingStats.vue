<template>
<div class="c-staking-stats">
    <div class="c-staking-stats__stats-container c-staking-stats__stats-container--global">
        <div
            v-for="{ label, value, unit, tooltip } in globalStats"
            :key="label"
            class="c-staking-stats__stat c-staking-stats__stat--global"
        >
            <div class="c-staking-stats__stat-label c-staking-stats__stat-label--global">
                {{ label }}
                <q-icon name="fas fa-info-circle" />
            </div>

            <div class="c-staking-stats__stat-value">
                {{ value }}
                <span class="c-staking-stats__stat-unit">{{ unit }}</span>
            </div>

            <q-tooltip
                :offset="[0, 56]"
                anchor="bottom middle"
                self="center middle"
            >
                <span class="u-text--pre">{{ tooltip }}</span>
            </q-tooltip>
        </div>
    </div>

    <q-card class="c-staking-stats__stats-container c-staking-stats__stats-container--personal">
        <div class="c-staking-stats__stat">
            <div class="c-staking-stats__stat-label">
                {{ personalStats.staked.label }}
                <q-icon name="fas fa-info-circle" />
            </div>

            <span class="c-staking-stats__stat-value">
                {{ personalStats.staked.value.stlos }}
                <span v-if="isLoggedIn" class="c-staking-stats__stat-unit c-staking-stats__stat-unit--personal">sTLOS</span>
                &#32; <!-- breaking space - avoid whitespace collapsing when this long stat wraps-->
            </span>
            <span v-if="isLoggedIn" class="c-staking-stats__stat-value">
                <wbr>
                &#8776; <!-- ≈ -->
                {{ personalStats.staked.value.tlos }}
                <span class="c-staking-stats__stat-unit c-staking-stats__stat-unit--personal">TLOS</span>
            </span>

            <q-tooltip
                :offset="[0, 56]"
                anchor="bottom middle"
                self="center middle"
            >
                <span class="u-text--pre">{{ personalStats.staked.tooltip }}</span>
            </q-tooltip>
        </div>
        <div class="c-staking-stats__stat">
            <div class="c-staking-stats__stat-label">
                {{ personalStats.unstaked.label }}
                <q-icon name="fas fa-info-circle" />
            </div>

            <span class="c-staking-stats__stat-value">
                {{ personalStats.unstaked.value }}
                <span v-if="isLoggedIn" class="c-staking-stats__stat-unit c-staking-stats__stat-unit--personal">
                    TLOS
                </span>
            </span>

            <q-tooltip
                :offset="[0, 56]"
                anchor="bottom middle"
                self="center middle"
            >
                <span class="u-text--pre">{{ personalStats.unstaked.tooltip }}</span>
            </q-tooltip>
        </div>
    </q-card>
</div>
</template>

<script>
import { fetchStlosApy, formatUnstakePeriod } from 'pages/staking/staking-utils';
import { formatWei, WEI_PRECISION } from 'src/lib/utils';
import { mapGetters } from 'vuex';

export default {
    name: 'StakingStats',
    props: {
        stlosContractInstance: {
            type: Object,
            required: true,
        },
        stlosBalance: {
            type: String,
            default: null,
        },
        stlosValue: {
            type: String,
            default: null,
        },
        totalUnstakedTlosBalance: {
            type: String,
            default: null,
        },
        unstakePeriodSeconds: {
            type: Number,
            default: null,
        },
    },
    data: () => ({
        stlosTvl: null,
        stlosApy: null,
    }),
    computed: {
        ...mapGetters('login', ['isLoggedIn']),
        globalStats() {
            return [{
                label: 'APY',
                value: this.stlosApy ?? '--',
                unit: '%',
                tooltip: 'APY: Annual Percentage Yield\n\nThe annual rate of return after taking compound interest into account.\n\n' +
                    'Interest is compounded approximately every 30 minutes. The percentage rate is not fixed, meaning that ' +
                    'it will change over time with the total amount of TLOS staked across Telos EVM and Native. ' +
                    'Rewards are disbursed from a community rewards pool into the sTLOS contract.',
            }, {
                label: 'TVL',
                value: this.formatWeiForStats(this.stlosTvl, true).replace(/\B(?=(\d{3})+(?!\d))/g, ' '),
                unit: 'TLOS',
                tooltip: 'TVL: Total Value Locked\n\nThe current value, in TLOS, of all assets held in the sTLOS ' +
                    '(Staked TLOS) smart contract, i.e. the sum of all TLOS staked on the Telos EVM at this moment.',
            }];
        },
        personalStats() {
            return {
                staked: {
                    label: 'Staked',
                    value: {
                        stlos: this.formatWeiForStats(this.stlosBalance),
                        tlos: this.formatWeiForStats(this.stlosValue),
                    },
                    tooltip: 'Staked\n\n' +
                        'The total staked amount associated with the logged-in account, i.e. ' +
                        'your sTLOS token balance, along with its value in TLOS',
                },
                unstaked: {
                    label: 'Unstaked',
                    value: this.formatWeiForStats(this.totalUnstakedTlosBalance),
                    tooltip: 'Unstaked\n\n' + // switch unstake to unstakesecondspretty
                        'The total value of TLOS which you have unstaked, both locked and unlocked.\n\n' +
                        'When you unstake\u2014i.e. redeem\u2014some value of sTLOS, the equivalent amount of ' +
                        `TLOS is sent into escrow ("locked") for ${this.unlockPeriodPretty}; during this time, ` +
                        'you cannot interact with this TLOS.\n\n' +
                        'After the unlock period has elapsed, you can claim your unlocked TLOS from the Claim tab ' +
                        'on this page, at which point it will be added to your account TLOS balance.',
                },
            };
        },
        unlockPeriodPretty() {
            return formatUnstakePeriod(this.unstakePeriodSeconds);
        },
    },
    async created() {
        await this.fetchGlobalStats();
    },
    methods: {
        async fetchGlobalStats() {
            try {
                this.stlosTvl = (await this.stlosContractInstance.totalAssets()).toString();
            } catch ({ message: tvlError }) {
                console.error(`Failed to fetch sTLOS TVL: ${tvlError}`);
                this.stlosTvl = null;
                this.stlosApy = null;

                return;
            }

            if (this.stlosTvl === null)
                return;

            try {
                this.stlosApy = await fetchStlosApy(this.$telosApi);
            } catch ({ message: apyError }) {
                console.error(`Failed to fetch sTLOS APY: ${apyError}`);
                this.stlosApy = null;
            }
        },
        formatWeiForStats(wei) {
            const format = val => formatWei(val, WEI_PRECISION, 3);

            return wei === null ? '--' : format(wei);
        },
    },
}
</script>

<style lang="scss">
.c-staking-stats {
    display: flex;
    flex-wrap: wrap;

    @media screen and (min-width: $breakpoint-md-min) {
        justify-content: flex-end;
    }

    @media screen and (min-width: $breakpoint-lg-min) {
        flex-wrap: nowrap;
        gap: 16px;
    }

    &__stats-container {
        height: min-content;

        &--global {
            flex-basis: 100%;

            display: flex;
            align-items: center;
            justify-content: flex-start;
            gap: 32px;
            margin-bottom: 32px;

            @media screen and (min-width: $breakpoint-md-min) {
                justify-content: flex-end;
                margin: 0;
                padding: 0 0 12px;
            }

            @media screen and (min-width: $breakpoint-lg-min) {
                flex-basis: auto;
                padding: 12px;
            }
        }

        &--personal {
            display: flex;
            align-items: baseline;
            justify-content: space-evenly;
            gap: 24px;
            padding: 12px;
            margin-bottom: 24px;

            @media screen and (min-width: $breakpoint-sm-min) {
                max-width: max-content;
                margin: 0 auto 24px;
            }

            @media screen and (min-width: $breakpoint-md-min) {
                margin: 0 0 24px;
            }

            @media screen and (min-width: $breakpoint-lg-min) {
                margin: 0;
            }
        }
    }

    &__stat {
        width: fit-content;
        @media screen and (min-width: $breakpoint-md-min) {
            width: max-content;
        }

        &--global {
            position: relative;

            &:not(:last-of-type)::after {
                position: absolute;
                top: 0;
                right: -17px;
                bottom: 0;
                margin: auto;

                height: 80%;
                width: 1px;

                content: '';
                border-radius: 4px;
                background-color: #8591FD;
            }
        }
    }

    &__stat-label {
        font-size: 14px;
        white-space: nowrap;
        display: flex;
        align-items: center;
        gap: 4px;

        &--global {
            color: $white;
        }

        &--personal {
            color: $dark;
        }
    }

    &__stat-unit {
        display: inline-block;
        font-size: 10px;
        color: $secondary;
        transform: translateX(-2px);
        vertical-align: super;

        &--personal {
            @at-root .body--light & {
                color: darken($secondary, 10%);
            }
        }
    }

    &__stat-value {
        font-size: 18px;
        color: $accent;
        white-space: nowrap;
    }
}
</style>
