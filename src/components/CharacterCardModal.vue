<template>
  <div class="character-modal-wrapper" :class="{ active, flipped: modalFlipped }" :style="characterPaletteCSS"
    @click.stop="hideModal">
    <div class="character-modal" @click.stop="() => { }">
      <div class="cover-image" :style="
        'background-image:url(\' ' + characterOrEmpty.seriesImage + ' \')'
      "></div>
      <div class="content">
        <div class="image">
          <img ref="charImgElement" :src="characterImage" @load="onImageLoad" crossOrigin="anonymous" />
          <Tilt :options="{ glare: true, reverse: true }">
            <div class="image-bg" :style="'background-image:url(\' ' + characterImage + ' \')'"></div>
          </Tilt>
        </div>
        <div class="text">
          <div class="name">
            {{ characterOrEmpty.name }}
          </div>
          <div class="series">
            {{ characterOrEmpty.series }}
          </div>
          <div class="tabs-wrapper">
            <div class="tabs-nav">
              <div class="tab-item" :class="{ active: currentTab == 0 }" @click="currentTab = 0">
                Détails
              </div>
              <div class="tab-item" :class="{ active: currentTab == 1 }" @click="currentTab = 1">
                Description
              </div>
            </div>
            <div class="tabs-container">
              <div class="tab-content" v-if="currentTab == 0">
                <div class="ttable">
                  <div class="trow" v-if="characterOrEmpty.gender">
                    <span class="tfield">Genre</span>
                    {{ characterOrEmpty.gender }}
                  </div>
                  <div class="trow" v-if="characterOrEmpty.age">
                    <span class="tfield">Âge</span> {{ characterOrEmpty.age }}
                  </div>
                  <div class="trow" v-if="characterOrEmpty.favourites">
                    <span class="tfield">Favoris</span>
                    {{ characterOrEmpty.favourites }}
                  </div>
                </div>
              </div>
              <div class="tab-content" v-if="currentTab == 1">
                <p class="description" v-html="descriptionHTML"></p>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="actions">
        <button @click.stop="reloadCharacter">Chercher</button>
        <button @click.stop="editFormStart">Modifier</button>
        <button @click.stop="hideModal">Fermer</button>
      </div>
    </div>
    <div class="character-modal-edit" @click.stop="() => { }">
      <div class="image">
        <img :src="editForm.image" />
      </div>
      <div class="form">
        <div class="group">
          <label>Nom</label>
          <input type="text" v-model="editForm.name" />
        </div>
        <div class="group">
          <label>Série</label>
          <input type="text" v-model="editForm.series" />
        </div>
        <div class="group">
          <label>Image</label>
          <input type="text" v-model="editForm.image" />
        </div>
      </div>
      <div class="actions">
        <button @click="saveCharacter">Save</button>
        <button>Supprimer</button>
        <button @click="modalFlipped = false">Annuler</button>
      </div>
    </div>
  </div>
</template>

<script>
// Library
import Tilt from "vanilla-tilt-vue";
import ColorThief from "colorthief";
import hsv from "rgb-hsv";
import MarkdownIt from "markdown-it";
// Models
import Character from "../models/character";
//
export default {
  components: {
    Tilt,
  },
  data() {
    return {
      active: false,
      character: new Character("", ""),
      characterPalette: [],
      modalShown: false,
      modalFlipped: false,
      loading: false,
      currentTab: 0,
      editForm: {
        name: "",
        series: "",
        image: "",
      },
    };
  },
  computed: {
    descriptionHTML() {
      if (this.character.description == null)
        return "Aucune description disponible.";
      const md = new MarkdownIt();
      return md
        .render(this.character.description)
        .replaceAll(
          "~!",
          '<span class="spoiler"><span class="spoiler-btn"></span><span class="spoiler-content">'
        )
        .replaceAll("!~", "</span>");
    },
    characterOrEmpty() {
      if (this.character == null) {
        return new Character("", "");
      } else {
        return this.character;
      }
    },
    characterImage() {
      if (this.character == null || this.character.name.length <= 0) return "";
      if (this.character.image.length <= 0)
        return (
          "https://via.placeholder.com/200x350?text=" +
          this.character.name.replaceAll(" ", "+")
        );
      return this.character.image + "?" + new Date().getTime();
    },
    characterPaletteCSS() {
      if (this.characterPalette == null || this.characterPalette.length <= 0)
        return `
      --dominant-color: hsl(100, 0%, 40%);
      --muted-color: hsl(200, 0%, 30%);
      --muted-color-light: hsl(200, 0%, 40%);
      `;

      let saturationMax = 0;
      let saturatedHue = 0;
      let saturationMin = 100;
      let mutedHue = 0;

      this.characterPalette.forEach((color) => {
        const hsvColor = hsv(color[0], color[1], color[2]);
        if (hsvColor[1] > saturationMax) {
          saturationMax = hsvColor[1];
          saturatedHue = hsvColor[0];
        }
        if (hsvColor[1] < saturationMin) {
          saturationMin = hsvColor[1];
          mutedHue = hsvColor[0];
        }
      });
      let genderColor = "0, 0%, 30%";
      if (this.character.gender == "Homme") {
        genderColor = "213, 100%, 50%";
      } else if (this.character.gender == "Femme") {
        genderColor = "337, 100%, 50%";
      } else if (
        this.character.gender != null &&
        this.character.gender.length > 0
      ) {
        genderColor = "280, 100%, 50%";
      }

      //   console.log("char palette", this.characterPalette);

      return `
      --dominant-color: hsl(${saturatedHue}, 100%, 40%);
      --dominant-color-light: hsl(${saturatedHue}, 100%, 50%);
      --muted-color: hsl(${mutedHue}, 100%, 30%);
      --muted-color-light: hsl(${mutedHue}, 100%, 40%);
      --gender-color: hsl(${genderColor});
      --color-1: rgb(${this.characterPalette[0][0]}, ${this.characterPalette[0][1]}, ${this.characterPalette[0][2]});
      --color-2: rgb(${this.characterPalette[1][0]}, ${this.characterPalette[1][1]}, ${this.characterPalette[1][2]});
      --color-3: rgb(${this.characterPalette[2][0]}, ${this.characterPalette[2][1]}, ${this.characterPalette[2][2]});
      --color-4: rgb(${this.characterPalette[3][0]}, ${this.characterPalette[3][1]}, ${this.characterPalette[3][2]});
      --color-5: rgb(${this.characterPalette[4][0]}, ${this.characterPalette[4][1]}, ${this.characterPalette[4][2]});
      `;
    },
  },
  methods: {
    deleteCharacter() {
      this.$emit("delete-character", this.character);
    },
    async selectDominantColor() {
      if (this.character == null) return;
      if (this.character.image.length <= 0) return;
      const charImgElement = this.$refs.charImgElement;
      //   console.log("getting palette from image", charImgElement);
      const colorThief = new ColorThief();
      this.characterPalette = await colorThief.getPalette(charImgElement, 5);
      //   console.log("character palette obtained", this.characterPalette);
    },
    async showModal(character) {
      this.character = character;
      if (this.character.image.length <= 0) {
        this.active = true;
      }
    },
    hideModal() {
      this.characterPalette = [];
      this.modalFlipped = false;
      this.active = false;
      this.character = new Character("", "");
    },
    // Save Character
    saveCharacter() {
      this.$emit("save-character", {
        character: this.character,
        data: this.editForm,
      });
      this.modalFlipped = false;
    },
    // Fetch Character data based on name
    // WARNING: If the name is ambiguous, the character may be
    // replaced by the wrong data (Same name, wrong character).
    async reloadCharacter() {
      // console.log("reloading character");
      // console.log(props.character);
      const c = new Character(this.character.name, this.character.series);
      const data = await c.FetchDataSearch();
      console.log(data);
      if (
        data != null &&
        data.media.edges.findIndex((edge) => {
          return c.CompareSeriesName(edge.node.title);
        }) != -1
      ) {
        // console.log(data);
        c.FillFromFetched(data);
        this.$emit("save-character", {
          character: this.character,
          data: c.toJson(),
        });
      }
    },
    // Edit Character
    editFormStart() {
      this.editForm = {
        name: this.character.name,
        series: this.character.series,
        image: this.character.image,
      };
      this.modalFlipped = true;
    },
    async onImageLoad() {
      //   console.log("image loaded");
      try {
        if (
          this.characterPalette == null ||
          this.characterPalette.length <= 0
        ) {
          if (this.character.image.length > 0) {
            await this.selectDominantColor();
          }
          this.active = true;
        } else {
          console.log("palette not empty");
        }
      } catch (e) {
        console.error(e);
      }
    },
  },
};
</script>

<style lang="scss">
.character-modal-wrapper {
  position: fixed;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  background-color: rgba(0 0 0 / 60%);
  z-index: 200;
  overflow: hidden;
  opacity: 0;
  transition: all 0.2s ease;
  pointer-events: none;
  backdrop-filter: blur(5px);

  &.active {
    opacity: 1;
    pointer-events: all;
  }

  .character-modal {
    background-color: #fff;
    position: relative;
    width: min(100vw, 600px);
    margin: calc((100vh - min(100vw, 600px) / 1.75) / 2) auto;
    aspect-ratio: 1.75;
    border-radius: 4px;
    overflow: hidden;
    box-shadow: 0px 0px 0px 1px rgba(0, 0, 0, 0.05),
      0px 5px 15px rgba(0, 0, 0, 0.5);
    transition: all 0.2s ease;
    backface-visibility: hidden;
    transform: rotateY(0deg);

    .cover-image {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100px;
      background-color: #eee;
      background-size: cover;
      background-position: center;
      opacity: 0.6;
      z-index: 190;

      &::after {
        content: "";
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background: linear-gradient(to bottom,
            rgba(255, 255, 255, 0) 0%,
            rgba(255, 255, 255, 1) 100%);
        backdrop-filter: blur(5px);
        z-index: 191;
      }
    }

    .content {
      margin-top: 100px;
      padding: 2em;
      display: flex;

      .image {
        margin-top: -100px;
        z-index: 199;
        position: relative;

        img {
          position: absolute;
          opacity: 0;
          pointer-events: none;
        }

        .image-bg {
          background-color: #eee;
          width: min(100vw, 160px);
          aspect-ratio: 1/1.75;
          border-radius: 4px;
          background-size: cover;
          background-position: center;
          position: relative;
        }
      }

      .text {
        margin-top: -100px;
        padding-left: 2em;
        z-index: 199;
        width: calc(100vw - 160px + 2em);

        .name {
          font-size: 32px;
          font-weight: 500;
          color: var(--dominant-color);
          text-shadow: 0px 1px 2px rgb(0 0 0 / 20%);
          font-family: Overpass;
          height: 48px;
          text-overflow: ellipsis;
          word-break: break-word;
          white-space: nowrap;
          overflow: hidden;
        }

        .series {
          font-size: 14px;
          font-weight: 300;
          margin-bottom: 1em;
        }

        .tabs-wrapper {
          height: calc(100% - 100px);

          .tabs-nav {
            display: flex;
            margin-bottom: 1em;

            .tab-item {
              padding: 0.4em 1em;
              border-bottom: 2px solid transparent;
              cursor: pointer;
              text-transform: uppercase;
              transition: all 0.2s ease;

              &.active {
                border-color: var(--dominant-color);
              }
            }
          }

          .tabs-container {
            .tab-content {
              .description {
                max-height: 128px;
                overflow: auto;
                font-weight: 300;
                line-height: 1.5em;

                .spoiler {
                  background-color: var(--bs-body-color);
                }
              }

              .ttable {
                display: flex;
                column-gap: 1em;
                font-weight: 100;

                .trow {
                  font-weight: 100;
                  width: 33.333%;

                  .tfield {
                    font-weight: 500;
                  }
                }
              }
            }
          }
        }
      }
    }

    .actions {
      position: absolute;
      bottom: 0.5em;
      text-align: right;
      right: 2em;
      display: flex;
      width: 60%;
      justify-content: flex-end;
      column-gap: 5px;
      z-index: 500;

      button {
        background-color: var(--muted-color) !important;

        &:hover {
          background-color: var(--muted-color-light) !important;
        }
      }
    }

    .close {
      position: absolute;
      top: 1em;
      right: 1em;
      cursor: pointer;
      z-index: 200;
      font-size: 14px;
      font-weight: 500;
      color: #fff;
      text-shadow: 0px 0px 1px rgba(0, 0, 0, 0.5);
      opacity: 0.5;
      transition: all 0.2s ease;

      &:hover {
        opacity: 1;
        text-shadow: 0px 0px 2px rgba(0, 0, 0, 0.5);
      }
    }
  }

  .character-modal-edit {
    background-color: #fff;
    position: absolute;
    top: 0;
    left: 50%;
    right: 50%;
    width: min(100vw, 600px);
    margin: calc((100vh - min(100vw, 600px) / 1.75) / 2) auto;
    aspect-ratio: 1.75;
    border-radius: 4px;
    overflow: hidden;
    box-shadow: 0px 0px 0px 1px rgba(0, 0, 0, 0.05),
      0px 5px 15px rgba(0, 0, 0, 0.5);
    transition: all 0.2s ease;
    transform: translate(-50%, 0) rotateY(180deg);
    backface-visibility: hidden;
    display: flex;

    .image {
      margin: 2em;
      width: min(100vw, 160px);
      background-size: cover;
      background-position: center;
      position: relative;

      img {
        width: 100%;
        border-radius: 4px;
      }
    }

    .form {
      padding: 2em 0;
      width: calc(100% - min(100vw, 160px) - 8em);

      .group {
        label {
          display: block;
          font-size: 14px;
          font-weight: 500;
          margin-top: 0.5em;
          color: rgb(0, 162, 255);
          text-transform: uppercase;
        }

        input {
          display: block;
          width: 100%;
          border-radius: 4px;
          border: 1px solid #ccc;
          padding: 0.5em;
          font-size: 14px;
          font-weight: 300;
          margin-bottom: 1.5em;
        }
      }
    }

    .actions {
      display: flex;
      column-gap: 5px;
      position: absolute;
      bottom: 0.5em;
      right: 1em;
    }
  }

  &.flipped {
    .character-modal {
      transform: rotateY(180deg);
    }

    .character-modal-edit {
      transform: translate(-50%, 0) rotateY(0deg);
    }
  }
}
</style>
