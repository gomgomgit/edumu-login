<template>
  <!--begin::Wrapper-->
  <div class="">
    <!--begin::Form-->
    <Form
      class="form w-100"
      id="kt_login_signin_form"
      @submit="onSubmitLogin"
      :validation-schema="login"
    >

      <div class="fv-row mb-6">
        <label class="form-label fs-6 fw-medium text-black-50">Nama Sekolah</label>
        <div>
          <el-select
            v-model="form.kode"
            filterable
            remote
            placeholder="Pilih nama sekolah"
            :remote-method="getSekolah"
            :loading="loading"
            :no-data-text="nodatatext"
            class="w-100"
            name="kode"
            @input="errors.kode = false"
          >
            <el-option
              v-for="item in sekolahOption"
              :key="item.sekolah_kode"
              :label="`${item.sekolah_nama} (${item.sekolah_kode})`"
              :value="item.sekolah_kode"
            />
          </el-select>
        </div>
        <!--end::Input-->
        <div class="fv-plugins-message-container" v-if="errors.kode">
          <div class="fv-help-block">
            Harap pilih sekolah
          </div>
        </div>
      </div>
      <!--end::Input group-->

      <!--begin::Input group-->
      <div class="fv-row mb-6">
        <!--begin::Label-->
        <label class="form-label fs-6 fw-medium text-black-50">Username</label>
        <!--end::Label-->

        <!--begin::Input-->
        <el-input
					v-model="form.username"
					type="text"
					placeholder="Username"
          @input="errors.username = false"/>

        <!--end::Input-->
        <div class="fv-plugins-message-container" v-if="errors.username">
          <div class="fv-help-block">
            Harap isi Username
          </div>
        </div>
      </div>
      <!--end::Input group-->

      <!--begin::Input group-->
      <div class="fv-row mb-6">
        <!--begin::Wrapper-->
        <label class="form-label fw-medium text-black-50 fs-6">Password</label>
        <!--end::Wrapper-->

        <!--begin::Input-->
        <el-input
					v-model="form.password"
					type="password"
					placeholder="Password"
          show-password
          @input="errors.password = false"/>
        <!--end::Input-->
        <div class="fv-plugins-message-container" v-if="errors.password">
          <div class="fv-help-block">
            Harap isi Password
          </div>
        </div>
        <div class="text-end mt-2">
          <span class="text-primary fw-bold cursor-pointer">
            Lupa Password?
          </span>
        </div>
      </div>
      <!--end::Input group-->

      <!--begin::Actions-->
      <div class="text-center mt-10">
        <!--begin::Submit button-->
        <button
          type="submit"
          ref="submitButton"
          id="kt_sign_in_submit"
          class="btn btn-lg btn-primary w-100 mb-5"
          style="border-radius: 50px;"
        >
          <span class="indicator-label"> Masuk </span>

          <span class="indicator-progress">
            Tunggu Sebentar...
            <span
              class="spinner-border spinner-border-sm align-middle ms-2"
            ></span>
          </span>
        </button>
        <!--end::Submit button-->
      </div>
      <!--end::Actions-->
    </Form>
  </div>
  <!--end::Wrapper-->
</template>

<script setup>
import { defineComponent, onMounted, reactive, ref } from "vue";
import { ErrorMessage, Field, Form } from "vee-validate";
import { Actions, Mutations } from "@/store/enums/StoreEnums";
import { useStore } from "vuex";
import { useRouter } from "vue-router";
import Swal from "sweetalert2/dist/sweetalert2.min.js";
import * as Yup from "yup";
import axios from "axios";
import QueryString from "qs";
import { useToast } from "vue-toast-notification";
import CustomModal from '@/components/modals/CustomModal.vue'
import { request } from '@/util';
import md5 from 'md5'
import { inject } from 'vue'

const store = useStore();
const router = useRouter();

const submitButton = ref();

const cryoptojs = inject('cryptojs')

const listSekolah = ref()

const form = reactive({
  kode: '',
  username: '',
  password: ''
})
const errors = reactive({
  kode: false,
  username: false,
  password: false
})

const modalCode = ref(false)
const sekolahKode = ref()
const sekolahOption = ref([])
const nodatatext = ref('Sekolah tidak ada')

//Form submit function
const onSubmitLogin = async (values) => {
  if (!form.kode) {errors.kode = true}
  if (!form.username) {errors.username = true}
  if (!form.password) {errors.password = true}
  if(Object.values(errors).includes(true)){return false}

  store.dispatch(Actions.LOGOUT);
  axios.post('https://apisekolah.edumu.id/v1prod/sekolah', QueryString.stringify({code: form.kode}))
    .then(res => {
      if (res.data.status) {
        postLogin(values, res.data.data.sekolahs[0])
      } else {
        useToast().error(res.data.text)
      }
    })
    .catch(err => {
      console.log(err)
      useToast().error(err.message)
    })
};

function getSekolah(query) {
  axios.post('https://apisekolah.edumu.id/v3prod/search', QueryString.stringify({keyword: query}))
  .then(res => {
    if (res.data.status) {
      sekolahOption.value = res.data.data?.schools
      nodatatext.value = 'Sekolah tidak ada'
    } else {
      sekolahOption.value = res.data.data?.schools
      nodatatext.value = 'Masukkan kata kunci minimal 3 karakter'
    }
  })
  .catch(err => {
    console.log(err)
    useToast().error(err.message)
  })

}

function postLogin(data, sekolah) {
  const formData = new FormData()
  formData.append('user_username', form.username)
  formData.append('user_kunci', form.password)
  

  const loginUrl = sekolah.sekolah_kode == process.env.VUE_APP_REVAMP_SCHOOL
    ? `${process.env.VUE_APP_REVAMP_API_URL}/singleLogin`
    : `${process.env.VUE_APP_API_URL}/${sekolah.sekolah_kode}/apischool/singleLogin`

  axios.post(loginUrl, formData)
  .then(res => {
    if (res.data.success) {
      var loginData = {...res.data.data, ...sekolah}

      var stringLoginData = QueryString.stringify(loginData)
      var encryptedData = cryoptojs.AES.encrypt(stringLoginData, "edumuv2").toString()


      if (loginData.user_level == 'administrator') {
        window.location.href = `${process.env.VUE_APP_CMS_SEKOLAH_URL}/#/sign-in-process?data=${encryptedData}`
      }
      if (loginData.user_level == 'guru') {
        window.location.href = `${process.env.VUE_APP_CMS_GURU_URL}/#/sign-in-process?data=${encryptedData}`
      }
      if (loginData.user_level == 'siswa') {
        window.location.href = `${process.env.VUE_APP_CMS_SISWAWALI_URL}/#/sign-in-process?data=${encryptedData}`
      }
        
    } else {
      useToast().error('Kombinasi Username dan Password tidak sesuai!')
    }
  }).catch(err => {
    console.log(err)
    useToast().error(res.data.message)
  })
}
</script>

<style scoped>
  #list-sekolah-wrapper{
    background: white;
    width: 100%;
    position: absolute;
    top: 0;
    left: 0;
    box-shadow: rgba(0, 0, 0, 0.1) 0px 2px 12px 0px;
    box-sizing: border-box;
  }
  .sekolah-option:hover {
    background: #eee;
    cursor: pointer;
  }
</style>
