<script setup lang="ts">
import {ref, computed} from 'vue';
import {useCustomizerStore} from '@/stores/customizer';
import axios from 'axios';
import Logo from '@/components/shared/Logo.vue';
import {md5} from 'js-md5';
import {useAuthStore} from '@/stores/auth';
import {useCommonStore} from '@/stores/common';
import {marked} from 'marked';

const customizer = useCustomizerStore();
let dialog = ref(false);
let accountWarning = ref(false)
let updateStatusDialog = ref(false);
const username = localStorage.getItem('user');
let password = ref('');
let newPassword = ref('');
let newUsername = ref('');
let status = ref('');
let updateStatus = ref('')
let releaseMessage = ref('');
let hasNewVersion = ref(false);
let botCurrVersion = ref('');
let dashboardHasNewVersion = ref(false);
let dashboardCurrentVersion = ref('');
let version = ref('');
let releases = ref([]);
let devCommits = ref([]);

let installLoading = ref(false);

let tab = ref(0);

let releasesHeader = [
  {title: '标签', key: 'tag_name'},
  {title: '发布时间', key: 'published_at'},
  {title: '内容', key: 'body'},
  {title: '源码地址', key: 'zipball_url'},
  {title: '操作', key: 'switch'}
];

// Form validation
const formValid = ref(true);
const passwordRules = [
  (v: string) => !!v || '请输入密码',
  (v: string) => v.length >= 8 || '密码长度至少 8 位'
];
const usernameRules = [
  (v: string) => !v || v.length >= 3 || '用户名长度至少3位'
];

// 显示密码相关
const showPassword = ref(false);
const showNewPassword = ref(false);

// 账户修改状态
const accountEditStatus = ref({
  loading: false,
  success: false,
  error: false,
  message: ''
});

const open = (link: string) => {
  window.open(link, '_blank');
};

// 账户修改
function accountEdit() {
  accountEditStatus.value.loading = true;
  accountEditStatus.value.error = false;
  accountEditStatus.value.success = false;

  // md5加密
  // @ts-ignore
  if (password.value != '') {
    password.value = md5(password.value);
  }
  if (newPassword.value != '') {
    newPassword.value = md5(newPassword.value);
  }
  axios.post('/api/auth/account/edit', {
    password: password.value,
    new_password: newPassword.value,
    new_username: newUsername.value ? newUsername.value : username
  })
      .then((res) => {
        if (res.data.status == 'error') {
          accountEditStatus.value.error = true;
          accountEditStatus.value.message = res.data.message;
          password.value = '';
          newPassword.value = '';
          return;
        }
        accountEditStatus.value.success = true;
        accountEditStatus.value.message = res.data.message;
        setTimeout(() => {
          dialog.value = !dialog.value;
          const authStore = useAuthStore();
          authStore.logout();
        }, 2000);
      })
      .catch((err) => {
        console.log(err);
        accountEditStatus.value.error = true;
        accountEditStatus.value.message = typeof err === 'string' ? err : '修改失败，请重试';
        password.value = '';
        newPassword.value = '';
      })
      .finally(() => {
        accountEditStatus.value.loading = false;
      });
}

function getVersion() {
  axios.get('/api/stat/version')
      .then((res) => {
        botCurrVersion.value = "v" + res.data.data.version;
        dashboardCurrentVersion.value = res.data.data?.dashboard_version;
        let change_pwd_hint = res.data.data?.change_pwd_hint;
        if (change_pwd_hint) {
          dialog.value = true;
          accountWarning.value = true;
          localStorage.setItem('change_pwd_hint', 'true');
        } else {
          localStorage.removeItem('change_pwd_hint');
        }
      })
      .catch((err) => {
        console.log(err);
      });
}

function checkUpdate() {
  updateStatus.value = '正在检查更新...';
  axios.get('/api/update/check')
      .then((res) => {
        hasNewVersion.value = res.data.data.has_new_version;

        if (res.data.data.has_new_version) {
          releaseMessage.value = res.data.message;
          updateStatus.value = '有新版本！';
        } else {
          updateStatus.value = res.data.message;
        }
        dashboardHasNewVersion.value = res.data.data.dashboard_has_new_version;
      })
      .catch((err) => {
        if (err.response.status == 401) {
          console.log("401");
          const authStore = useAuthStore();
          authStore.logout();
          return;
        }
        console.log(err);
        updateStatus.value = err
      });
}

function getReleases() {
  axios.get('/api/update/releases')
      .then((res) => {
        releases.value = res.data.data.map((item: any) => {
          item.published_at = new Date(item.published_at).toLocaleString();
          return item;
        })
      })
      .catch((err) => {
        console.log(err);
      });
}

function getDevCommits() {
  fetch('https://api.github.com/repos/Soulter/AstrBot/commits', {
    headers: {
      'Host': 'api.github.com',
      'Referer': 'https://api.github.com'
    }
  })
      .then(response => response.json())
      .then(data => {
        devCommits.value = data.map((commit: any) => ({
          sha: commit.sha,
          date: new Date(commit.commit.author.date).toLocaleString(),
          message: commit.commit.message
        }));
      })
      .catch(err => {
        console.log(err);
      });
}

function switchVersion(version: string) {
  updateStatus.value = '正在切换版本...';
  installLoading.value = true;
  axios.post('/api/update/do', {
    version: version,
    proxy: localStorage.getItem('selectedGitHubProxy') || ''
  })
      .then((res) => {
        updateStatus.value = res.data.message;
        if (res.data.status == 'ok') {
          setTimeout(() => {
            window.location.reload();
          }, 1000);
        }
      })
      .catch((err) => {
        console.log(err);
        updateStatus.value = err
      }).finally(() => {
    installLoading.value = false;
  });
}

function updateDashboard() {
  updateStatus.value = '正在更新...';
  axios.post('/api/update/dashboard')
      .then((res) => {
        updateStatus.value = res.data.message;
        if (res.data.status == 'ok') {
          setTimeout(() => {
            window.location.reload();
          }, 1000);
        }
      })
      .catch((err) => {
        console.log(err);
        updateStatus.value = err
      });
}

function toggleDarkMode() {
  customizer.SET_UI_THEME(customizer.uiTheme === 'PurpleThemeDark' ? 'PurpleTheme' : 'PurpleThemeDark');
}

getVersion();
checkUpdate();

const commonStore = useCommonStore();
commonStore.createEventSource(); // log
commonStore.getStartTime();

</script>

<template>
  <v-app-bar elevation="0" height="55">

    <v-btn v-if="useCustomizerStore().uiTheme==='PurpleTheme'" style="margin-left: 22px;" class="hidden-md-and-down text-secondary" color="lightsecondary" icon rounded="sm"
           variant="flat" @click.stop="customizer.SET_MINI_SIDEBAR(!customizer.mini_sidebar)" size="small">
      <v-icon>mdi-menu</v-icon>
    </v-btn>
    <v-btn v-else style="margin-left: 22px; color: var(--v-theme-primaryText); background-color: var(--v-theme-secondary)" class="hidden-md-and-down" icon rounded="sm"
           variant="flat" @click.stop="customizer.SET_MINI_SIDEBAR(!customizer.mini_sidebar)" size="small">
      <v-icon>mdi-menu</v-icon>
    </v-btn>
    <v-btn v-if="useCustomizerStore().uiTheme==='PurpleTheme'" class="hidden-lg-and-up text-secondary ms-3" color="lightsecondary" icon rounded="sm" variant="flat"
           @click.stop="customizer.SET_SIDEBAR_DRAWER" size="small">
      <v-icon>mdi-menu</v-icon>
    </v-btn>
    <v-btn v-else class="hidden-lg-and-up ms-3" icon rounded="sm" variant="flat"
           @click.stop="customizer.SET_SIDEBAR_DRAWER" size="small">
      <v-icon>mdi-menu</v-icon>
    </v-btn>

    <div style="margin-left: 16px; display: flex; align-items: center; gap: 8px;">
      <span style=" font-size: 24px; font-weight: 1000;">Astr<span style="font-weight: normal;">Bot</span>
      </span>
      <span style="font-size: 12px; color: var(--v-theme-secondaryText);">{{ botCurrVersion }}</span>
    </div>

    <v-spacer/>

    <div class="mr-4">
      <small v-if="hasNewVersion">
        AstrBot 有新版本！
      </small>
      <small v-else-if="dashboardHasNewVersion">
        WebUI 有新版本！
      </small>
    </div>

    <v-btn size="small" @click="toggleDarkMode();" class="text-primary mr-2" color="var(--v-theme-surface)"
           variant="flat" rounded="sm">
      <!-- 明暗主题切换按钮 -->
      <v-icon v-if="useCustomizerStore().uiTheme === 'PurpleThemeDark'">mdi-weather-night</v-icon>
      <v-icon v-else>mdi-white-balance-sunny</v-icon>
    </v-btn>

    <v-dialog v-model="updateStatusDialog" width="1000">
      <template v-slot:activator="{ props }">
        <v-btn size="small" @click="checkUpdate(); getReleases(); getDevCommits();" class="text-primary mr-2"
               color="var(--v-theme-surface)"
               variant="flat" rounded="sm" v-bind="props">
          更新
        </v-btn>
      </template>
      <v-card>
        <v-card-title>
          <span class="text-h5">更新 AstrBot</span>
        </v-card-title>
        <v-card-text>
          <v-container>
            <v-progress-linear v-show="installLoading" class="mb-4" indeterminate color="primary"></v-progress-linear>

            <div>
              <h1 style="display:inline-block;">{{ botCurrVersion }}</h1>
              <small style="margin-left: 4px;">{{ updateStatus }}</small>
            </div>

            <div
                style="background-color: #646cff24; padding: 16px; border-radius: 10px; font-size: 14px; max-height: 400px; overflow-y: auto;"
                v-html="marked(releaseMessage)" class="markdown-content">

            </div>

            <div class="mb-4 mt-4">
              <small>💡 TIP: 跳到旧版本或者切换到某个版本不会重新下载管理面板文件，这可能会造成部分数据显示错误。您可在 <a
                  href="https://github.com/Soulter/AstrBot/releases">此处</a>
                找到对应的面板文件 dist.zip，解压后替换 data/dist 文件夹即可。当然，前端源代码在 dashboard 目录下，你也可以自己使用
                npm install 和 npm build
                构建。</small>
            </div>

            <v-tabs v-model="tab">
              <v-tab value="0">😊 正式版</v-tab>
              <v-tab value="1">🧐 开发版(master 分支)</v-tab>
            </v-tabs>
            <v-tabs-window v-model="tab">

              <!-- 发行版 -->
              <v-tabs-window-item key="0" v-show="tab == 0">
                <v-btn class="mt-4 mb-4" @click="switchVersion('latest')" color="primary" style="border-radius: 10px;"
                       :disabled="!hasNewVersion">
                  更新到最新版本
                </v-btn>
                <div class="mb-4">
                  <small>`更新到最新版本` 按钮会同时尝试更新机器人主程序和管理面板。如果您正在使用 Docker
                    部署，也可以重新拉取镜像或者使用 <a
                        href="https://containrrr.dev/watchtower/usage-overview/">watchtower</a> 来自动监控拉取。</small>
                </div>

                <v-data-table :headers="releasesHeader" :items="releases" item-key="name">
                  <template v-slot:item.body="{ item }: { item: { body: string } }">
                    <v-tooltip :text="item.body">
                      <template v-slot:activator="{ props }">
                        <v-btn v-bind="props" rounded="xl" variant="tonal" color="primary" size="small">查看</v-btn>
                      </template>
                    </v-tooltip>
                  </template>
                  <template v-slot:item.switch="{ item }: { item: { tag_name: string } }">
                    <v-btn @click="switchVersion(item.tag_name)" rounded="xl" variant="plain" color="primary">
                      切换
                    </v-btn>
                  </template>
                </v-data-table>
              </v-tabs-window-item>

              <!-- 开发版 -->
              <v-tabs-window-item key="1" v-show="tab == 1">
                <div style="margin-top: 16px;">
                  <v-data-table
                      :headers="[{ title: 'SHA', key: 'sha' }, { title: '日期', key: 'date' }, { title: '信息', key: 'message' }, { title: '操作', key: 'switch' }]"
                      :items="devCommits" item-key="sha">
                    <template v-slot:item.switch="{ item }: { item: { sha: string } }">
                      <v-btn @click="switchVersion(item.sha)" rounded="xl" variant="plain" color="primary">
                        切换
                      </v-btn>
                    </template>
                  </v-data-table>
                </div>
              </v-tabs-window-item>

            </v-tabs-window>

            <h3 class="mb-4">手动输入版本号或 Commit SHA</h3>

            <v-text-field label="输入版本号或 master 分支下的 commit hash。" v-model="version" required
                          variant="outlined"></v-text-field>
            <div class="mb-4">
              <small>如 v3.3.16 (不带 SHA) 或 42e5ec5d80b93b6bfe8b566754d45ffac4c3fe0b</small>
              <br>
              <a href="https://github.com/Soulter/AstrBot/commits/master"><small>查看 master 分支提交记录（点击右边的
                copy
                即可复制）</small></a>
            </div>
            <v-btn color="error" style="border-radius: 10px;" @click="switchVersion(version)">
              确定切换
            </v-btn>

            <v-divider class="mt-4 mb-4"></v-divider>
            <div style="margin-top: 16px;">
              <h3 class="mb-4">单独更新管理面板到最新版本</h3>
              <div class="mb-4">
                <small>当前版本 {{ dashboardCurrentVersion }}</small>
                <br>

              </div>

              <div class="mb-4">
                <p v-if="dashboardHasNewVersion">
                  有新版本！
                </p>
                <p v-else="dashboardHasNewVersion">
                  已经是最新版本了。
                </p>
              </div>

              <v-btn color="primary" style="border-radius: 10px;" @click="updateDashboard()"
                     :disabled="!dashboardHasNewVersion">
                下载并更新
              </v-btn>
            </div>
          </v-container>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="blue-darken-1" variant="text" @click="updateStatusDialog = false">
            关闭
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-dialog v-model="dialog" persistent max-width="500">
      <template v-slot:activator="{ props }">
        <v-btn size="small" class="text-primary mr-4" color="var(--v-theme-surface)" variant="flat" rounded="sm" v-bind="props">
          <v-icon class="mr-1">mdi-account</v-icon>
          账户
        </v-btn>
      </template>
      <v-card class="account-dialog">
        <v-card-text class="py-6">
          <div class="d-flex flex-column align-center mb-6">
            <logo title="AstrBot 仪表盘" subtitle="修改账户"></logo>
          </div>
          <v-alert 
            v-if="accountWarning" 
            type="warning"
            variant="tonal"
            border="start"
            class="mb-4"
          >
            <strong>安全提醒:</strong> 请修改默认密码以确保账户安全
          </v-alert>

          <v-alert
            v-if="accountEditStatus.success"
            type="success"
            variant="tonal"
            border="start"
            class="mb-4"
          >
            {{ accountEditStatus.message }}
          </v-alert>

          <v-alert
            v-if="accountEditStatus.error"
            type="error"
            variant="tonal"
            border="start"
            class="mb-4"
          >
            {{ accountEditStatus.message }}
          </v-alert>

          <v-form v-model="formValid" @submit.prevent="accountEdit">
            <v-text-field
              v-model="password"
              :append-inner-icon="showPassword ? 'mdi-eye-off' : 'mdi-eye'"
              :type="showPassword ? 'text' : 'password'"
              label="当前密码"
              variant="outlined"
              required
              clearable
              @click:append-inner="showPassword = !showPassword"
              prepend-inner-icon="mdi-lock-outline"
              hide-details="auto"
              class="mb-4"
            ></v-text-field>

            <v-text-field
              v-model="newPassword"
              :append-inner-icon="showNewPassword ? 'mdi-eye-off' : 'mdi-eye'"
              :type="showNewPassword ? 'text' : 'password'"
              :rules="passwordRules"
              label="新密码"
              variant="outlined"
              required
              clearable
              @click:append-inner="showNewPassword = !showNewPassword"
              prepend-inner-icon="mdi-lock-plus-outline"
              hint="密码长度至少 8 位"
              persistent-hint
              class="mb-4"
            ></v-text-field>

            <v-text-field
              v-model="newUsername"
              :rules="usernameRules"
              label="新用户名 (可选)"
              variant="outlined"
              clearable
              prepend-inner-icon="mdi-account-edit-outline"
              hint="留空表示不修改用户名"
              persistent-hint
              class="mb-3"
            ></v-text-field>
          </v-form>
          
          <div class="text-caption text-medium-emphasis mt-2">
            默认用户名和密码均为 astrbot
          </div>
        </v-card-text>
        
        <v-divider></v-divider>
        
        <v-card-actions class="pa-4">
          <v-spacer></v-spacer>
          <v-btn
            v-if="!accountWarning"
            variant="tonal"
            color="secondary"
            @click="dialog = false"
            :disabled="accountEditStatus.loading"
          >
            取消
          </v-btn>
          <v-btn
            color="primary"
            @click="accountEdit"
            :loading="accountEditStatus.loading"
            :disabled="!formValid"
            prepend-icon="mdi-content-save"
          >
            保存修改
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-app-bar>
</template>

<style>
.markdown-content h1 {
  font-size: 1.3em;
}

.markdown-content ol {
  padding-left: 24px;
  /* Adds indentation to ordered lists */
  margin-top: 8px;
  margin-bottom: 8px;
}

.markdown-content ul {
  padding-left: 24px;
  /* Adds indentation to unordered lists */
  margin-top: 8px;
  margin-bottom: 8px;
}

.account-dialog .v-card-text {
  padding-top: 24px;
  padding-bottom: 24px;
}

.account-dialog .v-alert {
  margin-bottom: 20px;
}

.account-dialog .v-btn {
  text-transform: none;
  font-weight: 500;
  border-radius: 8px;
}

.account-dialog .v-avatar {
  transition: transform 0.3s ease;
}

.account-dialog .v-avatar:hover {
  transform: scale(1.05);
}
</style>