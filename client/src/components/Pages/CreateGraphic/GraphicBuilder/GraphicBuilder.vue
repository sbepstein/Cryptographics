<template>
    <div class="container">
        <div class="left">
            <Canvas :canvasData="canvasData"></Canvas>
        </div>
        <!-- FIRST SCREEN OF GRAPHIC BUILDER FLOW  -->
        <div v-if="!buyScreen" class="right">
            <div class="selected-asset-packs">
                <h1 class="small-title">
                    Selected asset packs
                </h1>
                <div class="pack-list">
                    <asset-box
                            v-for="(assetPack, index) in selectedAssetPacks"
                            :key="index"
                            :assetPack="assetPack"
                            :small="true"
                            color="#eee" />
                    <div @click="changeTab" class="add-more">
                        +
                    </div>
                </div>
            </div>

            <div class="controls">
                <div class="top-controls">
                    <cg-checkbox v-on:checked="(val) => canvasData.frame = val">Frame</cg-checkbox>
                    <cg-checkbox v-on:checked="toggleRatio">Aspect ratio 1:1</cg-checkbox>
                    <cg-button
                            :disabled="isCanvasDrawing"
                            @click="renderCanvas"
                            button-style="transparent">
                        Recompose
                    </cg-button>
                </div>
                <separator></separator>
                <div class="bottom-controls">
                    <!--<cg-button @click="buyImage">Submit</cg-button>-->
                    <h1 class="large-title">Ξ {{ displayPrice() }}</h1>
                    <cg-button @click="buyScreen = true">Next</cg-button>
                </div>
            </div>
        </div>
        <!-- END OF FIRST SCREEN OF GRAPHIC BUILDER FLOW  -->

        <!-- SECOND SCREEN OF GRAPHIC BUILDER FLOW  -->
        <div v-if="buyScreen" class="right">
            <div class="selected-asset-packs">
                <div class="final-pack-list">
                    <div class="final-pack-item" v-for="(assetPack, index) in selectedAssetPacks">
                        <div class="asset-pack-meta">
                            <h1 class="small-title">{{ assetPack.packName }}</h1>
                            <div class="small-title">Ξ {{ assetPack.price }}</div>
                        </div>
                        <asset-box
                                :key="index"
                                :assetPack="assetPack"
                                :small="true"
                                color="#eee" />
                    </div>
                </div>
            </div>

            <div class="controls">
                <div class="top-controls buy-screen">
                    <div class="small-title">Art Title</div>
                    <Input placeholder="0/20"/>
                </div>
                <separator></separator>
                <div class="bottom-controls buy-screen">
                    <div>
                        <cg-button :button-style="'transparent'">Download</cg-button>
                    </div>
                    <div class="separate-controls">
                        <h1 class="large-title">Ξ {{ displayPrice() }}</h1>
                        <cg-button @click="buyImage">Claim Token</cg-button>
                    </div>
                </div>
            </div>
        </div>
        <!-- END OF SECOND SCREEN OF GRAPHIC BUILDER FLOW  -->
    </div>
</template>

<script>
  import {
    pickTenRandoms,
    getLandingPacks,
    calculatePrice,
    calculateFirstSeed,
    convertSeed,
  } from 'services/ethereumService';
  import Canvas from './Canvas.vue';
  import * as utils from 'services/utils';
  import * as imageService from 'services/imageService';
  import * as ipfsService from 'services/ipfsService';
  import { mapGetters } from 'vuex';
  import { METAMASK_ADDRESS, USERNAME } from 'store/user-config/types';
  import { CANVAS_DRAWING } from 'store/canvas/types';
  import Input from '../../../Shared/UI/Input';

  export default {
    name: 'GraphicBuilder',
    components: {
      Input,
      Canvas
    },
    data: () => ({
      buyScreen: false,
      canvasData: {
        assets: [],
        ratio: '2:3',
        frame: false,
        noBottomFrame: false,
      },
      randomSeed: 0,
      iterations: 0,
      timestamp: new Date().getTime(),
      randomHashIds: pickTenRandoms(),
      imagePrice: 0,
      potentialAssets: [],
      allAssets: [],
      selectedPacks: [],
    }),
    computed: {
      ...mapGetters({
        userAddress: METAMASK_ADDRESS,
        username: USERNAME,
        isCanvasDrawing: CANVAS_DRAWING,
      })
    },
    props: ['selectedAssetPacks'],
    methods: {

      async buyImage() {
        let canvas = Canvas.methods.getCanvasElement();
        let image = canvas.toDataURL('image/png');
        let ipfsHash = await ipfsService.uploadFile(image.substr(22));
        let pot = this.selectedPacks.map(assetPack =>
          assetPack.assets.map(asset => parseInt(asset.id)))
          .reduce((a, b) => a.concat(b), []);

        let img = await imageService.createImage(
          this.randomHashIds,
          this.timestamp,
          this.iterations,
          pot,
          this.username,
          this.userAddress,
          this.imagePrice,
          ipfsHash);
      },
      async renderCanvas() {
        this.iterations++;
        if (window.sessionStorage.length > 0) {
          const landingPacks = getLandingPacks();
          this.selectedPacks = [...new Set([...this.selectedPacks, ...landingPacks.packs])];
        }
        this.selectedPacks = [...new Set([...this.selectedPacks, ...this.selectedAssetPacks])];
        console.log(this.selectedPacks);
        let pot = this.selectedPacks.map(assetPack =>
          assetPack.assets.map(asset => parseInt(asset.id)))
          .reduce((a, b) => a.concat(b), []);
        console.log(pot);
        this.canvasData.assets = await imageService.getFinalAssets(this.randomSeed, this.iterations, utils.encode(pot), this.allAssets);
        console.log('iteration: ' + this.iterations);
        let picked = [];
        for (let i = 0; i < this.canvasData.assets.length; i++) {
          picked.push(this.canvasData.assets[i].id);
        }
        let price = await calculatePrice(picked, this.userAddress);

        if (pot.length === 0) {
          this.imagePrice = 0;
        }
        this.imagePrice = parseFloat(price);
        console.log('PRICE : ' + this.imagePrice);
      },
      changeTab() {
        this.$emit('tabChange', 'picker');
      },
      toggleRatio() {
        if (this.canvasData.ratio === '1:1') return this.canvasData.ratio = '2:3';
        this.canvasData.ratio = '1:1';
      },
      displayPrice() {
        return web3.utils.fromWei(this.imagePrice.toString(), 'ether');
      }
    },
    async created() {
      if (window.sessionStorage.length > 0) {
        this.canvasData.ratio = '1:1';
        this.randomHashIds = JSON.parse(sessionStorage.getItem('randomHashIds'));
        this.timestamp = sessionStorage.getItem('timestamp');
        this.iterations = parseInt(sessionStorage.getItem('iterations'), 10);
        const firstSeed = await calculateFirstSeed(this.timestamp, this.randomHashIds);
        this.randomSeed = await convertSeed(firstSeed);
        console.log('Random hash ids: ' + this.randomHashIds);
        console.log('Random seed ' + this.randomSeed);
        console.log('Iterations: ' + this.iterations);
        console.log('Timestamp : ' + this.timestamp);
        await this.renderCanvas();
        window.sessionStorage.clear();
      }
      else {
        this.renderCanvas();
      }
    },
    async beforeCreate() {
      //If session storage is empty then we will generate new params
      if (window.sessionStorage.length == 0) {
        this.randomHashIds = pickTenRandoms();
        this.timestamp = new Date().getTime();
        this.iterations = 0;
        this.allAssets = await imageService.loadDataForAssets();
        this.randomSeed = await calculateFirstSeed(this.timestamp, this.randomHashIds);
        this.randomSeed = await convertSeed(this.randomSeed);
      }
    },

    watch: {
      selectedAssetPacks: async function () {
        this.selectedPacks = this.selectedAssetPacks;
        this.iterations = 0;
        this.timestamp = new Date().getTime();
        this.randomSeed = await calculateFirstSeed(this.timestamp, this.randomHashIds);
        this.randomSeed = await convertSeed(this.randomSeed);
      }
    }
  };
</script>

<style scoped lang="scss">
    .container {
        position: relative;
        .left {
            flex-shrink: 1;
        }
        .right {
            display: flex;
            flex-flow: column;
            justify-content: space-between;
            flex-grow: 1;
            margin-left: 50px;
            max-width: 400px;
            width: 100%;
        }
        @media screen and (max-width: 767px) {
            flex-direction: column;
            .left {
                margin-bottom: 30px;
                flex-grow: 1;
                width: 100%;
                display: flex;
                .canvas-wrapper {
                    flex: 1 0 100%;
                }
            }
            .right {
                margin: 0;
                flex-direction: column;
                align-items: flex-start;
                .controls {
                    margin: 30px 0;
                    align-items: flex-start;
                    .asset-controls {
                        flex-direction: row;
                        align-items: center;
                        margin-bottom: 30px;
                        .selected-asset-packs {
                            margin-left: 30px;
                            .asset {
                                margin-top: 0;
                            }
                        }
                    }
                }
            }
        }
    }

    .selected-asset-packs {

        .final-pack-list {
            display: flex;
            flex-direction: column;
            justify-content: flex-end;
            align-items: flex-end;
            max-height: 300px;
            overflow-y: auto;

            .final-pack-item {
                display: flex;
                margin-bottom: 20px;

                .asset-pack-meta {
                    display: flex;
                    align-items: flex-end;
                    justify-content: flex-end;
                    flex-direction: column;
                    margin-right: 15px;

                    .small-title {
                        margin-bottom: 10px;

                        &:last-child {
                            margin: 0;
                        }
                    }
                }
            }
        }

        .pack-list {
            display: flex;
            flex-wrap: wrap;

            .asset-box {
                margin-right: 20px;
                margin-bottom: 20px;
            }
            .add-more {
                height: 55px;
                width: 75px;
                border: 1px solid #949494;
                display: flex;
                align-items: center;
                justify-content: center;
                font-size: 30px;
                color: #949494;
                cursor: pointer;

                &:hover {
                    color: #000;
                    border: 1px solid #000;
                }
            }
        }

        .small-title {
            margin-bottom: 20px;
        }
    }

    .formats {
        font-size: 10px;
        color: #7c7c7c;
        display: flex;
        margin: 20px 0;
        justify-content: flex-end;
        .format {
            cursor: pointer;
            text-align: right;
            margin-right: 20px;
            &:first-of-type {
                .box {
                    width: 38px;
                    height: 38px;
                }
            }
            &:last-of-type {
                margin-right: 0;
                .box {
                    width: 28px;
                    height: 38px;
                }
            }
            .box {
                background-color: #fff;
                margin-bottom: 5px;
                &.selected {
                    background-color: #000;
                }
            }
        }
    }

    .controls {
        .asset-controls {
            margin-bottom: 100px;
            display: inline-flex;
            flex-direction: column;
            align-items: flex-end;
        }
        .top-controls {
            margin-bottom: 20px;

            .button {
                margin-top: 10px;
            }

            &.buy-screen {
                display: flex;
                flex-direction: column;
                align-items: flex-end;

                input {
                    width: 185px;
                    margin-top: 20px;
                }
            }
        }
        .bottom-controls {
            display: flex;
            align-items: center;
            justify-content: flex-end;
            padding-top: 20px;


            &.buy-screen {
                justify-content: space-between;

                .separate-controls {
                    display: flex;
                    justify-content: flex-end;
                }
            }

            .large-title {
                margin: 0 15px 0 0;
            }
        }
    }
</style>