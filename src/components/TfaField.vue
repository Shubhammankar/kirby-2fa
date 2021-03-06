<template>
  <div>
    <k-button
      v-if="!userHas2faAuthentication"
      icon="lock"
      @click="getSecret"
      :disabled="isNotTheCurrentUserPage"
    >Enable 2fa</k-button>
    <k-button
      v-else
      icon="unlock"
      theme="negative"
      @click="$refs.disableDialog.open()"
      :disabled="isNotTheCurrentUserPage"
    >Disable 2fa</k-button>
    <k-dialog ref="enableDialog" size="medium">
      <template v-if="auth">
        <k-text class="mg-b-lg" size="large">
          <strong>Enabling two factor authentication</strong>
        </k-text>
        <ol class="mg-b-lg o-list">
          <li>Open your authenticator app.</li>
          <li>Add an account and scan the QR code shown here.</li>
        </ol>
        <img :src="auth.qr" class="mg-b-lg" width="140" alt="QR code"/>
        <ol class="mg-b-lg o-list" start="3">
          <li>Enter the code generated by your authenticator app.</li>
        </ol>
        <k-text-field v-model="code" class="mg-b-md" name="code" label="The code shown in your app"></k-text-field>
        <k-box v-if="Kbox.text" v-bind="Kbox" class="mg-b-md"></k-box>
      </template>
      <template v-else>
        <k-icon class="loader" type="loader"></k-icon>
      </template>

      <template slot="footer">
        <k-button-group>
          <k-button icon="cancel" @click="$refs.enableDialog.close()">Cancel</k-button>
          <k-button v-if="auth" icon="check" theme="positive" @click="verifyCode">
            Verify code
            <template v-if="isLoading">…</template>
          </k-button>
        </k-button-group>
      </template>
    </k-dialog>

    <k-dialog ref="disableDialog">
      <k-text class="mg-b-md">Are you sure that you want to disable two factor authentication?</k-text>
      <template slot="footer">
        <k-button-group>
          <k-button icon="cancel" @click="$refs.disableDialog.close()">Cancel</k-button>
          <k-button icon="check" theme="negative" @click="disabletfa">
            Disable
            <template v-if="isLoading">…</template>
          </k-button>
        </k-button-group>
      </template>
    </k-dialog>
  </div>
</template>
<script>
export default {
  props: {
    label: String,
    userHas2faAuthentication: Boolean
  },
  data() {
    return {
      isLoading: false,
      auth: undefined,
      code: "",
      Kbox: {
        text: null,
        theme: "negative"
      }
    };
  },
  methods: {
    getSecret() {
      if (this.isNotTheCurrentUserPage) return;

      this.$refs.enableDialog.open();
      this.$api
        .post("kirby-2fa/secret")
        .then(res => {
          this.auth = res;
        })
        .catch(res => {});
    },

    verifyCode() {
      if (this.isNotTheCurrentUserPage) return;

      this.isLoading = true;
      this.Kbox.text = null;

      this.$api
        .post("kirby-2fa/verify", {
          code: this.code
        })
        .then(({ verify }) => {
          if (verify) {
            this.userHas2faAuthentication = verify;
            this.$refs.enableDialog.close();
          } else {
            this.Kbox.text = "Incorrect code";
          }

          this.isLoading = false;
          this.code = "";
        })
        .catch(res => {});
    },

    disabletfa() {
      if (this.isNotTheCurrentUserPage) return;
      this.isLoading = true;

      this.$api
        .post("kirby-2fa/disable")
        .then(({ disabled }) => {
          if (disabled) {
            this.userHas2faAuthentication = !disabled;
            this.$refs.disableDialog.close();
          }
          this.isLoading = false;
        })
        .catch(res => {});
    }
  },

  computed: {
    isNotTheCurrentUserPage() {
      return (
        this.$route.name === "User" &&
        this.$route.params.id != this.$store.state.user.current.id
      );
    }
  }
};
</script>

<style scoped>

.mg-b-md {
  margin-bottom: 10px;
}
.mg-b-lg {
  margin-bottom: 20px;
}

.loader {
  animation: Spin 0.9s linear infinite;
}

ol.o-list{
  padding-left: 1.5rem;
}

ol.o-list li {
  list-style: decimal;
  margin-bottom: 0.5rem;
}
</style>