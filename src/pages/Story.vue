<template>
  <div class="container" v-loading.body="loading">
    <header>
      <div>
        <router-link :to="{ name: 'album' }">
          <h1 class="logo" v-if="album">{{ album.title }}</h1>
        </router-link>
        <RenameRemoveDropdown @rename="renameAlbum" @remove="removeAlbum" :confirming-removal="confirmingRemoval"></RenameRemoveDropdown>
      </div>
      <el-dropdown @command="handleDropdownCommand" trigger="click">
        <el-button class="button add-story">
          Add Story <i class="el-icon-caret-bottom el-icon--right"></i>
        </el-button>
        <el-dropdown-menu slot="dropdown">
          <el-dropdown-item command="addImageStory">
            <i class="el-icon-picture"></i>
            <span>Image</span>
          </el-dropdown-item>
          <el-dropdown-item command="addYoutubeStory">
            <i class="el-icon-caret-right"></i>
            <span>Youtube</span>
          </el-dropdown-item>
        </el-dropdown-menu>
      </el-dropdown>
    </header>
    <div class="story-container">
      <div v-if="ftue" class="ftue">
        <p>
          No stories have been addded to this album yet. Start by <el-button type="text" @click="addImageStory">adding a story now</el-button>.
        </p>
      </div>
      <Story v-else-if="album" v-for="(story, index) in album.heritage" :key="story.id" :story="story" :album-id="album.id" class="story"
             @story-deleted="removeStory(index)" @loading-stopped="loading = false">
        {{ story.description }}
      </Story>
    </div>
  </div>
</template>

<script>
import * as api from '@/api/';
import Story from '@/components/story/Story';
import RenameRemoveDropdown from '@/components/RenameRemoveDropdown';
import CreateImageStoryModalContent from '@/components/story/CreateImageStoryModalContent';
import CreateYoutubeStoryModalContent from '@/components/story/CreateYoutubeStoryModalContent';

export default {
  components: {
    Story, RenameRemoveDropdown, CreateImageStoryModalContent, CreateYoutubeStoryModalContent
  },
  data() {
    return {
      album: null,
      loading: true,
      confirmingRemoval: false,
      description: '',
      file: null,
      ftue: false,
      storyUrl: '',
      imgStoryModal: null,
      youtubeStoryModal: null
    };
  },
  mounted() {
    api.getDefaultAlbum(this.$route.params.id).then((res) => {
      this.album = res.data.response;
      if (!this.album.heritage.length) {
        this.ftue = true;
        this.loading = false;
      }
    }).catch((err) => {
      console.log(err);
    });
  },
  methods: {
    handleDropdownCommand(method, ...args) {
      this[method](args);
    },
    setDescription(description) {
      this.description = description;
    },
    setFile(file) {
      this.file = file;
    },
    setUrl(url) {
      this.storyUrl = url;
    },
    addYoutubeStory() {
      this.youtubeStoryModal = this.$createElement(CreateYoutubeStoryModalContent, {
        on: {
          'description-updated': this.setDescription,
          'url-updated': this.setUrl
        }
      });
      this.$msgbox({
        title: `Add a new story to "${this.album.title}"`,
        message: this.youtubeStoryModal,
        showCancelButton: true,
        confirmButtonText: 'Add Story',
        cancelButtonText: 'Cancel'
      }).then((action) => {
        if (action !== 'confirm') return;
        if (!this.storyUrl) this.$message.error('The url field is required');
        let newStory;
        api.addStory(this.album.id, this.description)
          .then((res) => {
            newStory = res.data.response;
            return api.addYoutubeAssetToStory(this.album.id, res.data.response.id, this.storyUrl);
          })
          .then((res) => {
            newStory.asset_name = res.data.response.source;
            newStory.asset_type = res.data.response.type;
            this.youtubeStoryModal.componentInstance.clearContents();
            this.album.heritage.push(newStory);
            this.ftue = false;
          })
          .catch((err) => {
            console.log(err);
          });
      }).catch(() => {});
    },
    addImageStory() {
      this.imgStoryModal = this.$createElement(CreateImageStoryModalContent, {
        on: {
          'description-updated': this.setDescription,
          'file-chosen': this.setFile
        }
      });
      this.$msgbox({
        title: `Add a new story to "${this.album.title}"`,
        message: this.imgStoryModal,
        showCancelButton: true,
        confirmButtonText: 'Add Story',
        cancelButtonText: 'Cancel'
      }).then((action) => {
        if (action !== 'confirm') return;
        if (!this.description) this.$message.error('The description field is required');
        const formData = new FormData();
        formData.append('asset', this.file.raw);
        let newStory;
        api.addStory(this.album.id, this.description)
          .then((res) => {
            newStory = res.data.response;
            return api.addImageAssetToStory(this.album.id, res.data.response.id, formData);
          })
          .then((res) => {
            newStory.asset_name = res.data.meta.location;
            this.imgStoryModal.componentInstance.clearContents();
            this.album.heritage.push(newStory);
            this.ftue = false;
          })
          .catch((err) => {
            console.log(err);
          });
      }).catch(() => {});
    },
    renameAlbum() {
      this.$prompt('Name', `Rename album "${this.album.title}"`, {
        confirmButtonText: 'Rename',
        cancelButtonText: 'Cancel',
      }).then((confirmed) => {
        const albumTitle = confirmed.value;
        api.renameDefaultAlbum(this.album.id, albumTitle).then(() => {
          this.album.title = albumTitle;
          this.$message.success('Album renamed successfully');
        }).catch((err) => {
          console.log(err);
        });
      }).catch((err) => {
        console.log(err);
      });
    },
    removeAlbum() {
      if (!this.confirmingRemoval) {
        this.confirmingRemoval = true;
        return;
      }
      api.deleteDefaultAlbum(this.album.id).then(() => {
        this.$router.push('/');
      }).catch((err) => {
        console.log(err);
      });
    },
    removeStory(index) {
      this.album.heritage.splice(index, 1);
      this.$message.success('The story has been deleted');
      if (!this.album.heritage.length) {
        this.ftue = true;
      }
    }
  }
};
</script>

<style>
h1 {
  padding-right: 10px;
}

header a {
  color: inherit!important;
}

.story-container {
  display: flex;
  flex-wrap: wrap;
}

.story {
  flex-grow: 1;
  margin: 10px 0 0 20px;
  /* 2 items per row, - margin - border */
  width: calc(50% - 20px - 2px);
}

asset-upload .el-upload {
  border: 1px dashed #d9d9d9;
  border-radius: 6px;
  cursor: pointer;
  position: relative;
  overflow: hidden;
}

.asset-upload .el-upload:hover {
  border-color: #20a0ff;
}

.asset-upload-icon {
  font-size: 28px;
  color: #8c939d;
  width: 178px;
  height: 178px;
  line-height: 178px;
  text-align: center;
  border: 1px dashed #20a0ff;
  margin-bottom: 15px;
}

.asset {
  width: 178px;
  height: 178px;
  display: block;
}

.asset-upload {
  text-align: center;
}
</style>
