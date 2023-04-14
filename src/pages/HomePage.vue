<template>
  <q-page class="row items-center justify-evenly">
    <div class="q-pa-md">
      <q-card class="my-card">
        <img src="../assets/sos_img.png">

        <q-card-section>
          <div class="text-h5">SOS Member Registration</div>
          <div class="text-subtitle2">Agent name : TBS Solutions Sdn Bhd</div>
        </q-card-section>

        <q-card-section class="q-pt-none">


          <q-form @submit="onSubmit" class="q-gutter-md">
            <q-input v-model="name" label="Your name *" lazy-rules
              :rules="[val => val && val.length > 0 || 'Please type something']">

            </q-input>

            <q-input v-model="phoneNumber" label="Your phone number *" lazy-rules :rules="[
              val => val !== null && val !== '' || 'Please type your phone number',
            ]">
              <template v-slot:prepend>
                <q-select borderless v-model="countryCodeSelected" :options="countryCodeOptions" />

              </template>
            </q-input>

            <div class="row justify-center">
              <q-btn label="Submit" type="submit" color="primary" />
            </div>
          </q-form>
        </q-card-section>
      </q-card>



    </div>
  </q-page>

  <q-dialog v-model="dialogVisible">
    <q-card>
      <q-card-section>
        <div class="text-h6">Alert</div>
      </q-card-section>

      <q-card-section class="q-pt-none">
        Merchant not found.
      </q-card-section>

      <q-card-actions align="right">
        <q-btn flat label="OK" color="primary" v-close-popup />
      </q-card-actions>
    </q-card>
  </q-dialog>
</template>

<script setup lang="ts">
import axios from 'axios';
import { onMounted, reactive, ref } from 'vue';
import { useRoute } from 'vue-router';
import * as fastXmlParser from 'fast-xml-parser';
import { useQuasar } from 'quasar';

const route = useRoute();
const $q = useQuasar()

const countryCodeSelected = ref('');
let countryCodeOptions: string[] = [

]

const dialogVisible = ref(false);

const name = ref(null)
const phoneNumber = ref(null)

const ws = reactive({
  wsUrl: '',
  wsCodeCrypt: 'TBSSOS',
  caUid: 'tbssos1_devp',
  caPwd: '123456',
  appId: 'TbsSos.App',
  appCode: 'TBSSOS',
  merchantNo: '',
});

const exp = new RegExp('<[^>]*>', 'm');

onMounted(async () => {
  $q.loading.show()

  if (route.query.caUid) {
    ws.caUid = route.query.caUid as string;
  }
  if (route.query.caPwd) {
    ws.caPwd = route.query.caPwd as string;
  }
  if (route.query.merchantNo) {
    ws.merchantNo = route.query.merchantNo as string;
  } else {
    dialogVisible.value = true;
  }



  const loginPub = await axios
    .get(
      `https://tbs.tbsdns.com/ClientAcct.MainService/1_1/MainService.asmx/LoginPub?wsCodeCrypt=TBSCLIENTACCTWS&acctUid=${ws.caUid}&acctPwd=${ws.caPwd}&loginType=TBSSOS&misc=`
    );

  console.log(loginPub);
  var trimTags: string = loginPub.data.replace(exp, '');
  let convertResponse = trimTags
    .replaceAll('&lt;', '<')
    .replaceAll('&gt;', '>')
    .replaceAll('&#xD;', '')
    .replace(/'/g, "'");

  const parser = new fastXmlParser.XMLParser();
  let jsonObj = parser.parse(convertResponse);
  console.log(jsonObj);

  ws.wsUrl = jsonObj.string.LoginAcctInfo.LoginAcct.WsUrl.replaceAll(
    '_wsver_',
    '5_7'
  );

  const phoneCountryCodeList = await axios
    .get(
      `${ws.wsUrl}/webapi/GetPhoneCountryCodeList?wsCodeCrypt=TBSSOS&caUid=${ws.caUid}&caPwd=${ws.caPwd}&keywordSearch=&startIndex=-1&noOfRecords=-1`
    );

  countryCodeOptions = phoneCountryCodeList.data.Country.map((item: any) => item.phone_code);
  if (countryCodeOptions.length > 0) {
    countryCodeSelected.value = countryCodeOptions[0];
  }

  $q.loading.hide()
});

async function onSubmit() {
  $q.loading.show()
  const inviteMember = await axios
    .get(
      `${ws.wsUrl}/webapi/InviteMember?wsCodeCrypt=TBSSOS&caUid=${ws.caUid}&caPwd=${ws.caPwd}&memberNo=&appId=TbsSos.App&userId=3A3EBA638C&appCode=TBSSOS&nickName=${name.value}&invitePhoneCountryCode=${encodeURIComponent(countryCodeSelected.value)}&invitePhoneNo=${phoneNumber.value}&merchantNo=${ws.merchantNo}`
    );
  $q.loading.hide()
  $q.dialog({
    title: 'Info',
    message: inviteMember.data.Result[0].result
  })
};

</script>
