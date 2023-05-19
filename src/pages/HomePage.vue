<template>
  <q-page class="row items-center justify-evenly">
    <div class="q-pa-md">
      <q-card class="my-card">
        <img src="../assets/sos_img.png">

        <q-card-section>
          <div class="text-h5">SOS Member Sign Up</div>
          <div class="text-subtitle2">Agent name : {{ agentProfileName }}</div>
        </q-card-section>

        <q-card-section class="q-pt-none">


          <q-form @submit="onSubmit" class="q-gutter-md">
            <q-input input-class="text-left" v-model="name" label="Your name *" lazy-rules
              :rules="[val => val && val.length > 0 || 'Please type your name']">

            </q-input>

            <q-input v-model="phoneNumber" label="Your phone number *" lazy-rules :rules="[
              val => val !== null && val !== '' || 'Please type your phone number',
              val => /^[0-9]+$/.test(val) || 'Please enter a valid phone number'
            ]">
              <template v-slot:prepend>
                <q-select borderless v-model="countryCodeSelected" :options="countryCodeOptions" option-value="phone_code"
                  option-label="phone_code">
                  <template v-slot:option="scope">
                    <q-item v-bind="scope.itemProps">
                      <q-item-section>
                        <q-item-label>{{ scope.opt.phone_code }} {{ scope.opt.name }}</q-item-label>
                      </q-item-section>
                    </q-item>
                  </template>
                </q-select>
              </template>
            </q-input>

            <!-- <vue3-q-tel-input v-model:tel="tel" :dropdown-options="countryCodeOptions" /> -->


            <div class="row justify-center">
              <q-btn label="Sign Up" type="submit" color="primary" />
            </div>
          </q-form>
        </q-card-section>
      </q-card>

      <div class="row justify-center q-pt-md">
        <div>
          v.5.7.3
        </div>
      </div>

    </div>
  </q-page>
</template>

<script setup >
// import axios from 'axios';
import { onMounted, reactive, ref } from 'vue';
import { useRoute } from 'vue-router';
import * as fastXmlParser from 'fast-xml-parser';
import { useQuasar } from 'quasar';
import { api } from 'boot/axios'
// import Vue3QTelInput from 'vue3-q-tel-input'
// import 'vue3-q-tel-input/dist/vue3-q-tel-input.esm.css'

const route = useRoute();
const $q = useQuasar()
const agentProfileName = ref('');
const countryCodeSelected = ref('');
const countryCodeOptions = ref([])

const name = ref(null)
const phoneNumber = ref(null)

const ws = reactive({
  wsUrl: '',
  wsCodeCrypt: 'TBSSOS',
  caUid: 'sos_prod',
  caPwd: '1y3!@hjuujTT9nv',
  appId: 'TbsSos.App',
  appCode: 'TBSSOS',
  merchantNo: '',
  userId: '',
});

const exp = new RegExp('<[^>]*>', 'm');

onMounted(async () => {
  $q.loading.show()

  if (route.query.caUid) {
    ws.caUid = route.query.caUid;
  }
  if (route.query.caPwd) {
    ws.caPwd = route.query.caPwd;
  }
  if (route.query.merchantNo) {
    ws.merchantNo = route.query.merchantNo;
  }
  if (route.query.userId) {
    ws.userId = route.query.userId;
  }

  const loginPub = await api
    .get(
      `https://tbs.tbsdns.com/ClientAcct.MainService/1_1/MainService.asmx/LoginPub?wsCodeCrypt=TBSCLIENTACCTWS&acctUid=${ws.caUid}&acctPwd=${ws.caPwd}&loginType=TBSSOS&misc=`
    );

  var trimTags = loginPub.data.replace(exp, '');
  let convertResponse = trimTags
    .replaceAll('&lt;', '<')
    .replaceAll('&gt;', '>')
    .replaceAll('&#xD;', '')
    .replace(/'/g, "'");

  const parser = new fastXmlParser.XMLParser();
  let jsonObj = parser.parse(convertResponse);

  ws.wsUrl = jsonObj.string.LoginAcctInfo.LoginAcct.WsUrl.replaceAll(
    '_wsver_',
    '5_7'
  );

  if (ws.merchantNo == '') {
    const defaultAgent = await api
      .get(
        `${ws.wsUrl}/webapi/DefaultAgent?wsCodeCrypt=TBSSOS&caUid=${ws.caUid}&caPwd=${ws.caPwd}`
      );
    ws.merchantNo = defaultAgent.data.DefaultAgent[0].merchant_no;
  }


  const getAgentProfile = await api
    .get(
      `${ws.wsUrl}/webapi/GetAgentProfile?wsCodeCrypt=TBSSOS&caUid=${ws.caUid}&caPwd=${ws.caPwd}&dbcode=${ws.merchantNo}`
    );
  if (getAgentProfile.data == null) {
    $q.loading.hide()
    return
  }
  agentProfileName.value = getAgentProfile.data.Armaster[0].name;
  console.log(agentProfileName);

  const phoneCountryCodeList = await api
    .get(
      `${ws.wsUrl}/webapi/GetPhoneCountryCodeList?wsCodeCrypt=TBSSOS&caUid=${ws.caUid}&caPwd=${ws.caPwd}&keywordSearch=&startIndex=-1&noOfRecords=-1`
    );

  countryCodeOptions.value = phoneCountryCodeList.data.Country;//.map((item: any) => item.phone_code);
  console.log(countryCodeOptions)
  if (countryCodeOptions.value.length > 0) {
    countryCodeSelected.value = countryCodeOptions.value[0];
  }

  $q.loading.hide()
});


async function onSubmit() {
  $q.loading.show()
  const inviteMember = await api
    .get(
      `${ws.wsUrl}/webapi/InviteMember?wsCodeCrypt=TBSSOS&caUid=${ws.caUid}&caPwd=${ws.caPwd}&memberNo=&appId=TbsSos.App&userId=${ws.userId == '' ? ws.merchantNo : ws.userId}&appCode=TBSSOS&nickName=${name.value}&invitePhoneCountryCode=${encodeURIComponent(countryCodeSelected.value.phone_code)}&invitePhoneNo=${phoneNumber.value}&merchantNo=${ws.merchantNo}`
    );
  $q.loading.hide()
  $q.dialog({
    title: 'Invitation Info',
    message: inviteMember.data.Result[0].result
  }).onOk(() => {
    if ($q.platform.is.android) {
      window.location.href = 'https://play.google.com/store/apps/details?id=my.com.tbs.tbssos.app'
    } else if ($q.platform.is.ios) {
      window.location.href = 'https://apps.apple.com/my/app/sos-fastlane/id1292518804'
    } else if ($q.platform.is.desktop) {
      window.location.href = 'https://appgallery.huawei.com/app/C103352725'
    }
  })
};

</script>
