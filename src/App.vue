<template>
  <v-app>
    <v-main fluid>
      <v-card>
        <v-toolbar flat color="#0059b3" dark>
          <v-toolbar-title class="flex-grow-1">BBBread</v-toolbar-title>
          <v-btn icon @click="get_csv"
            ><v-icon>{{ mdiFileDownload }}</v-icon></v-btn
          >
          <LoginMenu class="flex-grow-0" @logout="logout" @login="login" />
        </v-toolbar>
        <v-tabs background-color="#f2f2f2">
          <v-tab>
            <v-icon left> {{ mdiListStatus }} </v-icon>
            Status
          </v-tab>
          <v-tab>
            <v-icon left> {{ mdiClockAlertOutline }} </v-icon>
            Logs
          </v-tab>
          <v-tab>
            <v-icon left> {{ mdiLightningBoltOutline }} </v-icon>
            Power Supplies
          </v-tab>
          <v-tab
            href="https://github.com/lnls-sirius/bbb-function/blob/master/src/scripts/AUTOCONFIG.xlsx/"
          >
            <v-icon left> {{ mdiFileTableBoxMultipleOutline }} </v-icon>
            Autoconfig
          </v-tab>
          <v-tab-item>
            <v-card flat>
              <StatusTab />
            </v-card>
          </v-tab-item>
          <v-tab-item>
            <v-card flat>
              <LogsTab />
            </v-card>
          </v-tab-item>
          <v-tab-item>
            <v-card flat>
              <PsTab />
            </v-card>
          </v-tab-item>
          <v-tab-item>
            <v-card flat>
              <v-card-title>Redirecting to GitHub...</v-card-title>
            </v-card>
          </v-tab-item>
        </v-tabs>
      </v-card>
    </v-main>
    <FooterBar />
    <ConfirmDialog ref="confirm" />
    <v-snackbar
      v-model="$store.state.snackbar"
      timeout="4000"
      color="white"
      light
    >
      {{ $store.state.message }}

      <template v-slot:action="{ attrs }">
        <v-btn icon text v-bind="attrs" @click="$store.commit('hide_snackbar')">
          <v-icon>{{ mdiClose }}</v-icon>
        </v-btn>
      </template>
    </v-snackbar>
  </v-app>
</template>

<script>
import { PublicClientApplication } from "@azure/msal-browser";
import StatusTab from "./components/StatusTab";
import LogsTab from "./components/LogsTab";
import FooterBar from "./components/FooterBar";
import ConfirmDialog from "./components/ConfirmDialog";
import LoginMenu from "./components/LoginMenu";
import PsTab from "./components/PsTab";
import {
  mdiListStatus,
  mdiClockAlertOutline,
  mdiLightningBoltOutline,
  mdiClose,
  mdiFileTableBoxMultipleOutline,
  mdiFileDownload,
} from "@mdi/js";

export default {
  name: "App",
  data() {
    return {
      mdiListStatus,
      mdiClockAlertOutline,
      mdiClose,
      mdiLightningBoltOutline,
      mdiFileTableBoxMultipleOutline,
      mdiFileDownload,
    };
  },

  components: {
    StatusTab,
    LogsTab,
    FooterBar,
    ConfirmDialog,
    LoginMenu,
    PsTab,
  },

  async created() {
    const msalInstance = new PublicClientApplication(
      this.$store.state.msalConfig
    );

    this.$store.commit("set_instance", msalInstance);
  },
  async mounted() {
    this.$root.$confirm = this.$refs.confirm.open;

    const accounts = this.$store.state.msalInstance.getAllAccounts();
    if (accounts.length == 0) {
      return;
    }
    accounts[0].initials = this.getInitials(accounts[0]);
    this.$store.commit("set_account", accounts[0]);
  },
  methods: {
    async login() {
      await this.$store.state.msalInstance
        .loginPopup({})
        .then(() => {
          const accounts = this.$store.state.msalInstance.getAllAccounts();
          accounts[0].initials = this.getInitials(accounts[0]);
          this.$store.commit("set_account", accounts[0]);
        })
        .catch((error) => {
          console.error(`Error during authentication: ${error}`);
        });
      this.$store.commit(
        "show_snackbar",
        `Logged in as ${this.$store.state.account.username}`
      );
    },
    async logout() {
      await this.$store.state.msalInstance.logout({}).catch((error) => {
        console.error(error);
      });
    },
    getInitials(account) {
      return account.name.split(" ")[0].substring(0, 1);
    },
    async get_csv() {
      const response = await this.send_command("beaglebones");
      const resp_json = await response.json();

      let csv = Object.keys(resp_json[0]).join(",");

      for (let beagle of resp_json) csv += "\n" + Object.values(beagle);

      var utf_arr = new Uint16Array(
        csv.split("").map(function (k) {
          return k.charCodeAt(0);
        })
      );

      const download = URL.createObjectURL(
        new Blob([utf_arr], { type: "text/csv;charset=UTF-16LE;" })
      );

      // This is hackish, but seems to be the best way without opening a new window
      let hidden_link = document.createElement("a");
      hidden_link.href = download;
      hidden_link.download = "beaglebones.csv";
      hidden_link.click();
    },
  },
};
</script>

<style scoped>
.v-main {
  background: rgb(1, 45, 87);
}
</style>
