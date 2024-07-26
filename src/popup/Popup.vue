<script setup lang="js">
import { ref, onMounted, reactive } from 'vue'

const token = ref('')
const state = reactive({
  emailTo: "",
  userName: "",
  successStatus: "",
  faliedStatus: "",
  myToken: "",
})

onMounted(() => {
  Login()
  setUpState()
})

const Login = () => {
  chrome.identity.getAuthToken({ interactive: true }, function(myToken) {
    if (myToken) {
      chrome.storage.sync.set({ googleAccessToken: myToken })
      getAccessToken()
      state.successStatus = "Success to get access token"
    } else {
      console.error('Failed to get access token')
    }
  })
}

const getAccessToken = async () => {
  let myToken = await chrome.storage.sync.get(['googleAccessToken'])
  state.myToken = myToken.googleAccessToken
}

const setUpState = async () => {
  let mySetting = await chrome.storage.sync.get(['emailTo', 'userName'])
  state.emailTo = mySetting.emailTo
  state.userName = mySetting.userName
}

const dateNow = () => {
  return new Date().toISOString().slice(0, 10);
}

const MakeEmail = (emailBodyAttendance) => {
  let now = dateNow()
  let emailTitle = "[CATW群組郵件] " + state.userName + " " + now + " 工作日報"
  let rawEmail =
    "To: " + state.emailTo +"\r\n" +
    "Subject: =?UTF-8?B?" + btoa(unescape(encodeURIComponent(emailTitle))) + "?=\r\n\r\n"
  rawEmail += `Hi All,\r\n\r\n${emailBodyAttendance}\r\n\r\n`;

  let encodedMessage = btoa(unescape(encodeURIComponent(rawEmail)))
  let reallyEncodedMessage = encodedMessage.replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')

  return reallyEncodedMessage
}

const GetAttendanceList = async () => {
  let now = dateNow()
  let url = `https://kinkan.appspot.com/api/v1/attendances/detail?date=${now}&checkLatest=false`
  try {
    let response = await fetch(url, {
      referrer: "https://kinkan.appspot.com/",
      referrerPolicy: "strict-origin-when-cross-origin",
      body: null,
      method: "GET",
      mode: "cors",
      credentials: "include"
    });
    if (!response.ok) {
      let json = await response.json()
      console.log(json)
      throw new Error(`Response status: ${response.status}`)
    }
    let json = await response.json()
    console.log(json)
    return json.manHours
  } catch (error) {
    console.error(error.message)
    state.faliedStatus = "Failed to Get Attendance List"
  }
  return []
}

const Send = async (emailBodyAttendance) => {
  let encodedEmail = MakeEmail(emailBodyAttendance)

  try {
    let response = await fetch("https://www.googleapis.com/gmail/v1/users/me/messages/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${state.myToken}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({raw: encodedEmail})
    });

    if (!response.ok) {
      let json = await response.json()
      console.log(json)
      throw new Error(`Response status: ${response.status}`)
    }
    let json = await response.json()
    console.log(json)
  } catch (error) {
    console.error(error.message)
    state.faliedStatus = `Failed to send, please to relogin`
    state.myToken = ""
  }
}

const SaveToStorage = async () => {
  state.successStatus = ""
  chrome.storage.sync.set({
    emailTo: state.emailTo,
    userName: state.userName
  })
  await new Promise(r => setTimeout(r, 500));
  state.successStatus = "Saved"
}

const GetAttendanceAndSend = async () => {
  state.successStatus = ""
  state.faliedStatus = ""
  let manHours = await GetAttendanceList()
  if (manHours.length <= 0) {
    state.successStatus = "No Attendance Found"
    return
  }

  let emailBodyAttendanceStart = ""
  let emailBodyAttendance = manHours.reduce(
    (accumulator, currentValue, currentIndex) => accumulator + getOrderName(currentIndex+1, currentValue),
    emailBodyAttendanceStart,
  )
  await Send(emailBodyAttendance)
  state.successStatus = "Sent"
}

const getOrderName = (startNumber, manHour) => {
  if (/^\d{6}$/.test(manHour.orderNumber)) {
    return `${startNumber}. ${manHour.orderNumber}: ${manHour.orderName} \r\n`
  } else {
    return `${startNumber}. ${manHour.orderNumber} \r\n`
  }
}
</script>

<template>
  <main>
    <h3>KK Send Mail</h3>
    <div v-if="!state.myToken" class="calc">
      <button @click="Login">Login</button>
    </div>
    <div class="calc">
      <h4>Mail To</h4>
      <input v-model="state.emailTo">
    </div>
    <div class="calc">
      <h4>User Name</h4>
      <input v-model="state.userName">
    </div>
    <div class="calc">
      <button @click="SaveToStorage">Save</button>
      <button @click="GetAttendanceAndSend">Send</button>
    </div>
    <a>{{state.successStatus}}</a>
    <a class="falied">{{state.faliedStatus}}</a>
  </main>
</template>

<style>
:root {
  font-family:
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    'Segoe UI',
    Roboto,
    Oxygen,
    Ubuntu,
    Cantarell,
    'Open Sans',
    'Helvetica Neue',
    sans-serif;

  color-scheme: light dark;
  background-color: #242424;
}

@media (prefers-color-scheme: light) {
  :root {
    background-color: #fafafa;
  }

  a:hover {
    color: #42b983;
  }
}

body {
  min-width: 20rem;
  color-scheme: light dark;
}

main {
  text-align: center;
  padding: 1em;
  margin: 0 auto;
}

h3 {
  color: #42b983;
  text-transform: uppercase;
  font-size: 1.5rem;
  font-weight: 200;
  line-height: 1.2rem;
  margin: 2rem auto;
}

.calc {
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 1rem;

  > button {
    font-size: 1rem;
    padding: 0.5rem 1rem;
    border: 1px solid #42b983;
    border-radius: 0.25rem;
    background-color: transparent;
    color: #42b983;
    cursor: pointer;
    outline: none;
    width: 5rem;
    margin: 0 1rem;
  }

  > button:hover {
    background-color: #42b983;
    color: #ffffff;
  }

  > button:active {
    background-color: #42b983;
    color: #ffffff;
    box-shadow: 0 5px #666;
    transform: translateY(4px);
  }

  > h4 {
    font-size: 1rem;
    margin: 0 1rem;
    width: 7rem;
  }

  > label {
    font-size: 1.5rem;
    margin: 0 1rem;
  }

  > input {
    font-size: 1rem;
    padding: 0.5rem 1rem;
    border: 1px solid #42b983;
    border-radius: 0.25rem;
    margin: 0 1rem;
    width: 15rem;
  }
}

a {
  font-size: 1rem;
  margin: 0.5rem;
  color: #cccccc;
  text-decoration: none;
}

.falied {
  color: #ff6600;
}

</style>
