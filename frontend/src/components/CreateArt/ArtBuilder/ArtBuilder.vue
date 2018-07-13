<template>
    <div class="art-builder-wrapper">
        <div class="left-group">
            <Canvas :canvasData="canvasData"></Canvas>
        </div>
        <div class="right-group">
            <div class="recreate" @click="renderCanvas">
                <button id="recreate" class="default-button no-background">Recreate</button>
            </div>
            <div class="controls">
                <button type="button" v-on:click="changeTab" class="default-button no-background">Select Asset Packs
                </button>
                <div class="selected-asset-packs">
                    <div class="asset-pack-circle small selected" v-for="asset in selectedAssetPacks">
                        {{asset.id}}
                    </div>
                </div>
                <div class="formats">
                    <div class="first-format" v-on:click="setRatio('1:1')">
                        <div class="box"></div>
                        1:1
                    </div>
                    <div class="second-format" v-on:click="setRatio('2:3')">
                        <div class="box"></div>
                        2:3
                    </div>
                </div>
                <div class="frame">
                    <label>Frame <input v-model="canvasData.frame" type="checkbox"/> </label>
                </div>
                <button class="default-button submit" @click="buyImage">Submit</button>
            </div>
        </div>
    </div>
</template>

<script>
    import Canvas from '../../Canvas.vue';

    import IPFS from 'ipfs';
    import * as utils from '../../../../scripts/utils';
    import * as functions from '../../../../scripts/functions';
    import * as methods from '../../../methods';
    import {getAccounts} from '../../../../scripts/helpers';
    import * as ipfsService from '../../../../scripts/ipfsService';

    export default {
        name: 'art-builder',
        components: {Canvas},
        data: () => ({
            metamask_account: 0,
            canvasData: {
                assets: [],
                ratio: '1:1',
                frame: false,
            },
            random_seed: 0,
            iterations: 0,
            timestamp: new Date().getTime(),
            random_hash_ids: functions.pickTenRandoms(),
            image_price: 0,
            potential_assets: [],
            all_assets: [],
            packs_from_home : [],
        }),
        props: ['selectedAssetPacks'],

        methods: {

            async buyImage() {
                let canvas = Canvas.methods.getCanvasElement();
                let image = canvas.toDataURL('image/png');
                console.log(image);
                let ipfsHash = await ipfsService.uploadFile(image.substr(22));
                let pot = this.selectedAssetPacks.map(assetPack =>
                    assetPack.data.map(asset => parseInt(asset.id)))
                    .reduce((a, b) => a.concat(b), []);
                console.log(pot);

                console.log('RANDOM HASHES: ' + this.random_hash_ids);
                console.log('TIMESTAMP: ' + this.timestamp);
                console.log('ITERATIONS: ' + this.iterations);
                console.log('POTENTIAL ASSETS: ' + pot);
                console.log('MM ACCOUNT: ' + this.metamask_account);
                console.log('IMAGE PRICE: ' + this.image_price);
                let img = await methods.createImage(this.random_hash_ids, this.timestamp, this.iterations - 1, pot, 'Madjar', this.metamask_account, this.image_price, ipfsHash);
                console.log(img);
                // this.timestamp = new Date().getTime();
                // this.random_seed = await functions.calculateFirstSeed(this.timestamp, this.random_hash_ids);
                // this.iterations = 0;

            },
            async renderCanvas() {
                let pot = 0;
                if(window.sessionStorage.length > 0 ){
                    let selectedPacks = functions.generatePacks();
                    pot = selectedPacks.map(assetPack =>
                        assetPack.data.map(asset => parseInt(asset.id)))
                        .reduce((a, b) => a.concat(b), []);
                    window.sessionStorage.clear();
                } else {
                    pot = this.selectedAssetPacks.map(assetPack =>
                        assetPack.data.map(asset => parseInt(asset.id)))
                        .reduce((a, b) => a.concat(b), []);
                }
                console.log("POTENTIAL ASSETS: " + pot);
                console.log("RANDOM SEED: " + this.random_seed);
                console.log("ITERATIONS: " + this.iterations);
                console.log("TIMESTAMP: " + this.timestamp);

                this.canvasData.assets = await methods.getData(this.random_seed, this.iterations, utils.encode(pot), this.all_assets);
                this.iterations++;
                let picked = [];
                for (let i = 0; i < this.canvasData.assets.length; i++) {
                    picked.push(this.canvasData.assets[i].id);
                }
                let price = await functions.calculatePrice(picked, this.metamask_account);
                if (pot.length === 0) {
                    this.image_price = 0;
                }
                this.image_price = parseInt(price, 10);
                console.log("PRICE : " + this.image_price);

            },
            changeTab() {
                this.$emit('tabChange', 'picker');
            },
            setRatio(ratio) {
                this.canvasData.ratio = ratio;
            }
        },
        async beforeCreate() {
            if(window.sessionStorage.length == 0) {
                this.random_hash_ids = functions.pickTenRandoms();
                this.timestamp = new Date().getTime();
                this.metamask_account = await getAccounts();
                this.iterations = 0;
                this.all_assets = await methods.loadDataForAssets();
                this.random_seed = await functions.calculateFirstSeed(this.timestamp, this.random_hash_ids);
                this.random_seed = await functions.convertSeed(this.random_seed);
            }
            window.node = new IPFS({
                repo: 'cryptographics',
                config: {
                    Bootstrap: ipfsService.bootstrapNodes,
                    Addresses: {
                        Swarm: [],
                    },
                }
            });

        },

        async created() {
            if (window.sessionStorage.length > 0) {
                console.log("ENTERED");
                this.random_hash_ids = JSON.parse(sessionStorage.getItem("random_hash_ids"));
                this.timestamp = sessionStorage.getItem("timestamp");
                this.iterations = sessionStorage.getItem("iterations");
                console.log(this.random_hash_ids);
                this.random_seed = await functions.calculateFirstSeed(this.timestamp, this.random_hash_ids);
                this.random_seed = await functions.convertSeed(this.random_seed);

                console.log("Random seed " + this.random_seed);
                console.log("Iterations: " + this.iterations);
                console.log("Timestamp : " + this.timestamp);

                await this.renderCanvas()
            }
        },
        watch: {
                selectedAssetPacks: async function () {
                    window.sessionStorage.clear();
                    this.random_hash_ids = functions.pickTenRandoms();
                    this.iterations = 0;
                    this.timestamp = new Date().getTime();
                    this.random_seed = await functions.calculateFirstSeed(this.timestamp, this.random_hash_ids);
                    this.random_seed = await functions.convertSeed(this.random_seed);
                }
            }
    };
</script>

<style scoped lang="scss">
    .art-builder-wrapper {
        display: flex;
        height: 100%;
        position: relative;
        justify-content: space-between;

    .left-group {
        flex-shrink: 1;

    img {
        height: 100%;
    }

    }

    .right-group {
        box-sizing: border-box;
        padding: 0 30px;
        display: flex;
        align-items: flex-end;
        justify-content: space-between;
        flex-grow: 1;
        min-width: 300px;

    .recreate {
    }

    }

    .selected-asset-packs {
        margin-bottom: 20px;

    .asset-pack-circle {
        margin: 20px 12px 0;

    &
    :last-child {
        margin-right: 0;
    }

    }

    }

    .formats {
        font-size: 10px;
        color: #7c7c7c;
        display: flex;
        margin: 20px 0;
        justify-content: flex-end;

    .first-format, .second-format {
        text-align: right;
        cursor: pointer;
    }

    .first-format {

    .box {
        width: 38px;
        height: 38px;
        background-color: #fff;
        margin-bottom: 5px;
    }

    margin-right:

    20
    px

    ;
    }
    .second-format {

    .box {
        width: 28px;
        height: 38px;
        background-color: #000;
        margin-bottom: 5px;
    }

    }
    }

    .frame {
        font-size: 10px;
        color: #7c7c7c;
        text-align: right;
    }

    .default-button.submit {
        background-color: #000;
        color: #fff;
        float: right;
        margin-top: 20px;
    }

    @media all and (max-width: 1280px) {
        .right-group {

        .recreate {
            margin-right: 50px
        }
    }

    }
    }
</style>