<template>
  <v-app :style="{ backgroundImage: `url(${bgImage})`, backgroundAttachment: 'fixed', backgroundSize: 'cover', backgroundPosition: 'center', backgroundRepeat: 'no-repeat' }">
    <v-container class="profile-container">
      <v-card class="elevation-16 gradient-card" style="padding: 30px" flat>
        <v-row>
          <v-col md="4" xs="12" class="picture-section">
            <div class="picture-wrapper">
              <img class="profile-picture" src="https://q.qlogo.cn/headimg_dl?dst_uin=1302057949&spec=640&img_type=webp" alt="Avatar" />
            </div>
          </v-col>
          <v-col md="8" xs="12" class="info-section">
            <div class="name-container">
              <h1 class="profile-name">
                <span class="no-wrap">梦清 | Akari Rin</span>
              </h1>
            </div>
            <div class="label-section">
              <div class="label-header">语言</div>
              <v-chip size="small" label prepend-icon="mdi-translate">简体中文</v-chip>
              <v-chip size="small" label prepend-icon="mdi-alphabetical-variant">English</v-chip>
            </div>
            <div class="label-section">
              <div class="label-header">技术栈</div>
              <v-chip size="small" label prepend-icon="mdi-language-cpp" class="chip-cpp">C++</v-chip>
              <v-chip size="small" label prepend-icon="mdi-cloud" class="chip-cloudflare">Cloudflare</v-chip>
              <v-chip size="small" label prepend-icon="mdi-docker" class="chip-docker">Docker</v-chip>
              <v-chip size="small" label prepend-icon="mdi-language-java" class="chip-java">Java</v-chip>
              <v-chip size="small" label prepend-icon="mdi-language-javascript" class="chip-javascript">JavaScript</v-chip>
              <v-chip size="small" label prepend-icon="mdi-kubernetes" class="chip-kubernetes">Kubenetes</v-chip>
              <v-chip size="small" label prepend-icon="mdi-database" class="chip-mysql">MySQL</v-chip>
              <v-chip size="small" label prepend-icon="mdi-database" class="chip-postgresql">PostgreSQL</v-chip>
              <v-chip size="small" label prepend-icon="mdi-language-python" class="chip-python">Python</v-chip>
              <v-chip size="small" label prepend-icon="mdi-database" class="chip-redis">Redis</v-chip>
              <v-chip size="small" label prepend-icon="mdi-language-java" class="chip-spring">Spring</v-chip>
              <v-chip size="small" label prepend-icon="mdi-language-typescript" class="chip-typescript">TypeScript</v-chip>
              <v-chip size="small" label prepend-icon="mdi-vuejs" class="chip-vue">Vue</v-chip>
            </div>
            <div class="label-section">
              <div class="label-header">开发工具</div>
              <v-chip size="small" label class="chip-clion">
                <template v-slot:prepend>
                  <img :src="CLionIcon" class="ide-icon" />
                </template>
                CLion
              </v-chip>
              <v-chip size="small" label class="chip-datagrip">
                <template v-slot:prepend>
                  <img :src="DataGripIcon" class="ide-icon" />
                </template>
                DataGrip
              </v-chip>
              <v-chip size="small" label class="chip-idea">
                <template v-slot:prepend>
                  <img :src="IntelliJIDEAIcon" class="ide-icon" />
                </template>
                IntelliJ IDEA
              </v-chip>
              <v-chip size="small" label class="chip-pycharm">
                <template v-slot:prepend>
                  <img :src="PyCharmIcon" class="ide-icon" />
                </template>
                PyCharm
              </v-chip>
              <v-chip size="small" label prepend-icon="mdi-microsoft-visual-studio-code" class="chip-vscode">VSCode</v-chip>
              <v-chip size="small" label class="chip-webstorm">
                <template v-slot:prepend>
                  <img :src="WebStormIcon" class="ide-icon" />
                </template>
                WebStorm
              </v-chip>
            </div>
            <div class="label-section">
              <div class="label-header">操作系统</div>
              <v-chip size="small" label prepend-icon="mdi-microsoft-windows" class="chip-windows">10</v-chip>
              <v-chip size="small" label prepend-icon="mdi-microsoft-windows" class="chip-windows">11</v-chip>
              <v-chip size="small" label prepend-icon="mdi-apple" class="chip-ios">iOS</v-chip>
              <v-chip size="small" label prepend-icon="mdi-apple" class="chip-ipados">iPadOS</v-chip>
              <v-chip size="small" label prepend-icon="mdi-debian" class="chip-debian">Debian 13</v-chip>
            </div>
          </v-col>
        </v-row>
      </v-card>
      <v-card class="elevation-16 mt-6 mb-6" flat>
        <v-card-title class="section-title">
          <v-icon class="mr-2" size="small">mdi-greater-than</v-icon>
          Who am I?
        </v-card-title>
        <v-card-text>
          <v-row>
            <v-col md="6" xs="12">
              <p class="section-paragraph">
                也许我们该重新认识一下了，你好，我叫梦清
              </p>
              <p class="section-paragraph">
                WIP
              </p>
            </v-col>
            <v-col md="6" xs="12">
              <v-list>
                <v-list-item class="section-list-header">
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-tag</v-icon>
                  </template>
                  关于我
                </v-list-item>
                <v-list-item
                  href="mailto:akaririn1028@gmail.com"
                  target="_blank"
                  class="identity-link"
                >
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-gmail</v-icon>
                  </template>
                  <span class="label">邮箱</span>
                  <span class="ml-2 fake-link">akaririn1028@gmail.com</span>
                </v-list-item>
                <v-list-item
                  href="https://akari/blog"
                  target="_blank"
                  class="identity-link"
                >
                  <template v-slot:prepend>
                    <!-- TODO:这里的图标换成博客的favicon -->
                    <v-icon size="small">mdi-tag</v-icon>
                  </template>
                  <span class="ml-2 fake-link">梦清の小窝</span>
                </v-list-item>
              </v-list>
              <v-list>
                <v-list-item class="section-list-header">
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-open-source-initiative</v-icon>
                  </template>
                  开源
                </v-list-item>
                <v-list-item
                  href="https://github.com/AkariRin"
                  target="_blank"
                  class="identity-link"
                >
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-github</v-icon>
                  </template>
                  <span class="label">GitHub</span>
                  <span class="ml-2 fake-link">梦清</span>
                </v-list-item>
                <v-list-item
                  href="https://wakatime.com/@akari"
                  target="_blank"
                  class="identity-link"
                  style="display: flex; align-items: center;"
                >
                  <img
                    src="https://wakatime.com/badge/user/40464278-56b3-4816-a5f6-c6bd962b7279.svg?style=for-the-badge"
                    style="display: block;"
                  />
                </v-list-item>
                <v-list-item
                  href="https://hub.docker.com/u/akaririn"
                  target="_blank"
                  class="identity-link"
                >
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-docker</v-icon>
                  </template>
                  <span class="label">Docker Hub</span>
                  <span class="ml-2 fake-link">Akari Rin</span>
                </v-list-item>
                <v-list-item
                  href="https://pypi.org/user/akaririn/"
                  target="_blank"
                  class="identity-link"
                >
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-language-python</v-icon>
                  </template>
                  <span class="label">PyPI</span>
                  <span class="ml-2 fake-link">Akari Rin</span>
                </v-list-item>
                <v-list-item
                  href="https://www.npmjs.com/~akaririn"
                  target="_blank"
                  class="identity-link"
                >
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-npm</v-icon>
                  </template>
                  <span class="label">npm</span>
                  <span class="ml-2 fake-link">Akari Rin</span>
                </v-list-item>
              </v-list>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>
      <v-card class="elevation-16 mt-6 mb-6" flat>
        <v-card-title class="section-title">
          <v-icon class="mr-2" size="small">mdi-controller</v-icon>
          NO GAME NO LIFE
        </v-card-title>
        <v-card-text>
          <v-row>
            <v-col md="6" xs="12">
              <p class="section-paragraph">
                WIP
              </p>
            </v-col>
            <v-col md="6" xs="12">
              <v-list>
                <v-list-item class="section-list-header">
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-account</v-icon>
                  </template>
                  平台
                </v-list-item>
                <v-list-item
                  href="https://steamcommunity.com/id/akaririn1028/"
                  target="_blank"
                  class="identity-link"
                >
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-steam</v-icon>
                  </template>
                  <span class="label">Steam</span>
                  <span class="ml-2 fake-link">二阶堂希罗</span>
                </v-list-item>
                <v-list-item class="identity-link">
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-minecraft</v-icon>
                  </template>
                  <span class="label">Minecraft</span>
                  <span class="ml-2">Akari1028</span>
                </v-list-item>
              </v-list>
              <v-list>
                <v-list-item class="section-list-header">
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-pistol</v-icon>
                  </template>
                  FPS
                </v-list-item>
                <v-list-item class="identity-link">
                  <span class="label">瓦罗兰特<small>国服</small></span>
                  <span class="ml-2">二阶堂希罗#81329</span>
                </v-list-item>
                <v-list-item class="identity-link">
                  <span class="label">完美对战平台</span>
                  <span class="ml-2">梦清</span>
                </v-list-item>
                <v-list-item
                  href="https://arena.5eplay.com/data/player/akari_vjn4f5"
                  target="_blank"
                  class="identity-link"
                >
                  <span class="label">5E对战平台</span>
                  <span class="ml-2 fake-link">Akari梦清</span>
                </v-list-item>
              </v-list>
              <v-list class="mt-4">
                <v-list-item class="section-list-header">
                  <template v-slot:prepend>
                    <v-icon size="small">mdi-tag</v-icon>
                  </template>
                  二游
                </v-list-item>
                <v-list-item class="identity-link">
                  <span class="label">明日方舟<small>国服</small></span>
                  <span class="ml-2">ユメスミカゲ#1812</span>
                </v-list-item>
              </v-list>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>
      <v-card class="elevation-16 mt-2 mb-6" flat>
        <v-card-text>
          <!-- TODO:这里的图标换成Akari Network的logo -->
          <v-btn
            variant="text"
            href="https://akari.moe/"
            target="_blank"
            prepend-icon="mdi-server-network"
            class="mb-2"
          >
            Running on Akari Network
            <span class="link ml-2">//akari.moe/</span>
          </v-btn>
          <v-btn
            variant="text"
            href="https://status.akari.moe/"
            target="_blank"
            prepend-icon="mdi-server-network"
            class="mb-2"
          >
            <span class="link ml-2">Service Status/</span>
          </v-btn>
          <v-divider class="my-2"></v-divider>
          <div class="copyright-text">
            Copyright © 2018-2025 Akari Rin. 插图均有其独立版权许可。
          </div>
        </v-card-text>
      </v-card>
    </v-container>
  </v-app>
</template>

<script setup lang="ts">
import bgImage from '@/assets/bg.jpg';
import CLionIcon from '@/assets/icons/CLion_icon.svg';
import DataGripIcon from '@/assets/icons/DataGrip_icon.svg';
import IntelliJIDEAIcon from '@/assets/icons/IntelliJ_IDEA_icon.svg';
import PyCharmIcon from '@/assets/icons/PyCharm_icon.svg';
import WebStormIcon from '@/assets/icons/WebStorm_icon.svg';
</script>

<style scoped>

.profile-container {
  max-width: 900px;
  margin: 0 auto;
  padding: 20px;
}

.gradient-card {
  background: linear-gradient(135deg, rgba(255, 0, 51, 0.7) 0%, rgba(255, 105, 180, 0.7) 50%, rgba(255, 255, 255, 0.6) 100%) !important;
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.3);
}

.picture-section {
  display: flex;
  justify-content: center;
  align-items: flex-start;
}

.picture-wrapper {
  padding: 20px;
}

.profile-picture {
  display: block;
  width: 200px;
  height: 200px;
  border: 8px solid white;
  border-radius: 50%;
  overflow: hidden;
  box-shadow: 0 0 15px rgba(0, 0, 0, 0.3);
}

.profile-picture img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.profile-picture a {
  display: block;
}

.info-section {
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
}

.name-container {
  margin-bottom: 20px;
}

.profile-name {
  font-size: 2.5rem;
  font-weight: 300;
  line-height: 1.25em;
  margin: 0;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.25);
}

.no-wrap {
  display: inline-block;
}

.label-section {
  margin-bottom: 20px;
  margin-top: 15px;
}

.label-header {
  font-size: 0.8em;
  font-weight: 600;
  letter-spacing: 1px;
  text-transform: uppercase;
  color: rgba(255, 255, 255, 0.9);
  margin-bottom: 8px;
}

.section-title {
  font-size: 1.5rem;
  font-weight: 300;
  letter-spacing: 0.02em;
  color: #333;
  display: flex;
  align-items: center;
}

.section-paragraph {
  color: #555;
  font-size: 1.1rem;
  font-weight: 300;
  line-height: 1.75em;
  margin: 1em 0;
}

.section-list-header {
  font-weight: 600;
  color: #666;
  border-bottom: 1px solid rgba(0, 0, 0, 0.05);
  padding-bottom: 8px;
}

.identity-link {
  color: #333;
  text-decoration: none;
  transition: background-color 0.25s;
}

.identity-link:hover {
  background-color: rgba(1, 105, 124, 0.05);
}


.label {
  font-weight: 500;
}

.fake-link {
  color: #0d5f70;
  font-weight: 500;
}

.copyright-text {
  color: #aaa;
  font-size: 0.9rem;
  line-height: 2em;
  text-align: center;
}

:deep(.v-chip) {
  margin: 4px;
}

:deep(.ide-icon) {
  width: 1.2em;
  height: 1.2em;
  object-fit: contain;
  margin-right: 0.5em;
}

:deep(.chip-cpp) {
  background-color: #00599C !important;
  color: white !important;
}

:deep(.chip-java) {
  background-color: #007396 !important;
  color: white !important;
}

:deep(.chip-javascript) {
  background-color: #F7DF1E !important;
  color: #333 !important;
}

:deep(.chip-python) {
  background-color: #3776AB !important;
  color: white !important;
}

:deep(.chip-typescript) {
  background-color: #3178C6 !important;
  color: white !important;
}

:deep(.chip-vue) {
  background-color: #42B883 !important;
  color: white !important;
}

:deep(.chip-spring) {
  background-color: #6DB33F !important;
  color: white !important;
}

:deep(.chip-clion) {
  background-color: #087CFA !important;
  color: white !important;
}

:deep(.chip-datagrip) {
  background-color: #21D789 !important;
  color: white !important;
}

:deep(.chip-idea) {
  background-color: #F97603 !important;
  color: white !important;
}

:deep(.chip-pycharm) {
  background-color: #087CFA !important;
  color: white !important;
}

:deep(.chip-vscode) {
  background-color: #007ACC !important;
  color: white !important;
}

:deep(.chip-webstorm) {
  background-color: #087CFA !important;
  color: white !important;
}

:deep(.chip-windows) {
  background-color: #0078D4 !important;
  color: white !important;
}

:deep(.chip-debian) {
  background-color: #FCC624 !important;
  color: #333 !important;
}

:deep(.chip-kubernetes) {
  background-color: #326CE5 !important;
  color: white !important;
}

:deep(.chip-ios) {
  background-color: #000000 !important;
  color: white !important;
}

:deep(.chip-ipads) {
  background-color: #000000 !important;
  color: white !important;
}

:deep(.chip-cloudflare) {
  background-color: #F38020 !important;
  color: white !important;
}

:deep(.chip-docker) {
  background-color: #2496ED !important;
  color: white !important;
}

:deep(.chip-mysql) {
  background-color: #00758F !important;
  color: white !important;
}

:deep(.chip-postgresql) {
  background-color: #336791 !important;
  color: white !important;
}

:deep(.chip-redis) {
  background-color: #DC382D !important;
  color: white !important;
}

@media (max-width: 768px) {
  .profile-name {
    font-size: 1.8rem;
  }

  .profile-picture {
    width: 150px;
    height: 150px;
  }

  .section-paragraph {
    font-size: 1rem;
  }

}
</style>
