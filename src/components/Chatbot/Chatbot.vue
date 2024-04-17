<script setup>
import $ from "jquery";
import { Swiper, SwiperSlide } from "swiper/vue";
import "swiper/css";
import "swiper/css/pagination";
import { Navigation } from "swiper/modules";
import { ref, onMounted, watch, reactive } from "vue";

function isWordPress() {
  // Check for common WordPress classes in the body element
  if (document.body.classList.contains('wordpress')) {
    return true;
  }


  // Check for WordPress-specific scripts or stylesheets
  const scripts = document.querySelectorAll('script[src*="wp-includes"]');
  const styles = document.querySelectorAll('link[href*="wp-content"]');
  if (scripts.length > 0 || styles.length > 0) {
    return true;
  }

  // Check for WordPress-specific meta tags
  const metaGenerator = document.querySelector('meta[name="generator"][content*="WordPress"]');
  if (metaGenerator) {
    return true;
  }

  return false;
}
function getBaseUrl() {
  // Get the current URL
  const currentURL = window.location.href;

  // Split the URL by slashes
  const urlParts = currentURL.split('/');

  // Construct the base URL
  const baseUrl = urlParts[0] + '//' + urlParts[2];

  return baseUrl;
}



let socket = false;
// const socket_url="ws://localhost:6014?botname=Aria&token=8374"
// const socket_url="wss://chatbot.snaketest.gq/?botname=Aria&token=8374"
const socket_url="wss://shopperassist.hmcg.com/?botname=Aria&token=8374"
// const socket_url="ws://44.216.26.81:6014/?botname=Aria&token=8374"
if (localStorage.getItem("token") || true) {
  //  socket = new WebSocket('wss://chatbot.fidelity-me.com?botname=Aria&token=8374'+localStorage.getItem('token'));
  socket = new WebSocket(
    socket_url + localStorage.getItem("token")
  );
}
const botimage = ref("");
const header = ref("");
const sendImg = ref("");


const myinput = ref(null);
const loading = ref(false);
const mute = ref(true);
const question = ref("");

const BotMessageTime = () => {
  const options = { weekday: "short", hour: "numeric", minute: "numeric", hour12: true };
  const date = new Date();
  const formattedDate = date.toLocaleString("en-US", options);
  return formattedDate;
};
let firstMessage = { role: "system", content: "Welcome to Toys For Pets! search for Toys for your Pets! 🐾 , click below to explore!, Don't see your pet? Send a friendly Hi!", time: BotMessageTime() }
let Messages = reactive([]);
const Context = reactive([]);



if (isWordPress()) {

  const baseUrl = getBaseUrl();
  const apiEndpoint = '/wp-json/toy-chat-bot/v1/options';

  // Construct the full API URL
  const apiUrl = baseUrl + apiEndpoint;
  
  // Make the API request
  fetch(apiUrl)
    .then(response => response.json())
    .then(data => {
      console.log('API Response:', data);
      const {
        chat_bot_image,
        heading_text,
        welcome_message,
        send_message_icon,
        color_scheme
      } = data
      if(chat_bot_image){
        botimage.value = chat_bot_image
      }
      if(heading_text){
        header.value = heading_text
      }
      if(send_message_icon){
        sendImg.value = send_message_icon
      }
      if(color_scheme){
        document.querySelector('body').style.setProperty('--color', color_scheme);
      }
      if(chat_bot_image){
        document.querySelector('body').style.setProperty('--cm-bot-image', `url(${chat_bot_image})`);
      }
      if(welcome_message){
        Messages.shift()
        Context.shift()
        firstMessage = { ...firstMessage, content: welcome_message }
        Messages.unshift(firstMessage)
        Context.unshift({...firstMessage,time:undefined})
      }
    })
    .catch(error => {
      console.error('API Error:', error);
    });


}




const MessagesWatchman = (allmsgs) => {
  const message = allmsgs[allmsgs.length - 1];
  if (message?.role == "manual") {
    Context.push({ role: "system", content: message.content });
  }
  if (
    message?.role != "system" &&
    message?.role != "flight_booking" &&
    message?.role != "hotel_booking"
  ) {
    $(".chat-logs")
      .stop()
      .animate({ scrollTop: $(".chat-logs")[0].scrollHeight }, 1000);
  }
};
watch(Messages, MessagesWatchman);
watch(question, () => {
  if (!socket) {
    if (localStorage.getItem("token") || true) {
      socket = new WebSocket(
        socket_url +
          localStorage.getItem("token")
      );
    }
    connect();
  }
});
watch(
  () => loading.value,
  (newval) => {
    if (!newval) {
      setTimeout(() => {
        myinput.value.focus();
      });
    }
  }
);
const modules = reactive([Navigation]);
function Ask(Question){
  question.value=Question
  askQuestion()
}
function askQuestion() {
  let ques = question.value;
  question.value = "";
  let time = BotMessageTime();
  let msg = { role: "user", content: ques };
  if (ques) {
    Messages.push({ ...msg, time: time });
    Context.push(msg);
    if (socket) {
      socket.send(JSON.stringify(Context));
    } else {
      let time = BotMessageTime();
      Messages.push({
        role: "manual",
        content: "You are not authorized for communication",
        time: time,
      });
    }
  }
}
let msgMindex = ref(null);
let msgCindex = ref(null);
let speachstring = false;

function defaultMessage(){
  Messages.push(firstMessage)
  Context.push({...firstMessage,time:undefined})
  Messages.push({ role:"default",content:"default", time: BotMessageTime() });
}

function getProduct(toy){
  let time = BotMessageTime();
  Messages.push({
    role: "toy",
    content: toy,
    time: time,
  });
}

async function connect() {
  if (socket) {
    socket.onopen = function () {
      console.log("Socket open.");
    };
    socket.onmessage = function (message) {
      message = JSON.parse(message.data);
      switch (message.role) {
        case "system":
          if (message.start) {
            let msg = { role: "system", content: "" };
            let time = BotMessageTime();
            Messages.push({ ...msg, time: time ,uuid:message.uuid});
            Context.push(msg);
            msgCindex.value = JSON.parse(JSON.stringify(Context)).length - 1;
            msgMindex.value = JSON.parse(JSON.stringify(Messages)).length - 1;
            speachstring = "";
            loading.value = true;
          } else if (message.end) {
            msgMindex.value = null;
            msgCindex.value = null;
            loading.value = false;
            if (mute.value && speachstring) {
              methodModel.textToVoice(speachstring);
              speachstring = false;
            }
            $(".chat-logs")
              .stop()
              .animate({ scrollTop: $(".chat-logs")[0].scrollHeight }, 1000);
          } else {
            const mindex = Messages.findIndex(m=>m.uuid===message.uuid)
            // console.log(mindex,message.uuid)
            let concatMessage = {
              ...Messages[mindex],
              content: Messages[mindex]?.content + message.content,
            };
            let concatContext = {
              ...Context[msgCindex.value],
              content: Context[msgCindex.value]?.content + message.content,
            };
            Context.pop();
            Messages.splice(mindex,1,concatMessage);
            Context.push(concatContext);
            const element = document.querySelector(".chat-logs")
            if (element.scrollHeight - element.scrollTop === element.clientHeight) {
                // console.log('Scrolled to the bottom');
                $(".chat-logs")
                .stop()
                .animate({ scrollTop: $(".chat-logs")[0].scrollHeight }, 50);
                // Perform your desired action when scrolled to the bottom
            }
          }
          break;
        case "toys_search":
        if (message.start) {
          loading.value = true;
        }else{
          let time = BotMessageTime();
          if(message.content.length){
            Messages.push({
              role: "manual",
              content: "Here is a list of toys for your pet",
              time: time,
            });
            MessagesWatchman(Messages);
            let msg = {
              role: "toys_search",
              content: message.content,
              type: "outbound",
            };
            Messages.push({ ...msg, time: time });
            loading.value = false;
          }else{
            Messages.push({
              role: "manual",
              content: "Sorry no Products found related to your search.",
              time: time,
            });
          }
        }
          break;
        default:
          console.log(message);
          break;
      }
    };
    socket.onerror = function (error) {
      console.error("WebSocket error:", error);
      socket = false;
    };
    socket.onclose = function (event) {
      socket = new WebSocket(socket_url);
      connect();
      // console.log('WebSocket connection refresh pending. ',socket);
    };
  }
}
connect();
async function startOver(){
  Messages.length=0
  Context.length=0
  socket = new WebSocket(
    socket_url + localStorage.getItem("token")
  );
  await connect()
  defaultMessage()
}
  onMounted(() => {
  // methodModel.textToVoice(" ");
  defaultMessage()
  $("#chat-circle").click(function () {
    $("#chat-circle").toggle("scale");
    $(".chat-box").toggle("scale");
  });

  $(".chat-box-toggle").click(function () {
    $("#chat-circle").toggle("scale");
    $(".chat-box").toggle("scale");
  });
  // $(".chat-logs").stop().animate({ scrollTop: $(".chat-logs")[0].scrollHeight}, 10000);
});
</script>


<template>
     <div>
    <div id="chat-circle" class="btn btn-raised">
      <img v-if="botimage" :src="botimage" class="img-fluid">
      <img v-else src="@/assets/images/WinstonIcon512.png" class="img-fluid">
    </div>
    <div class="chat-box">
      <div class="chat-box-header">
        <div class="chat_box_header_logo d-flex align-items-center">
          <a href="">
            <img v-if="botimage" :src="botimage" class="img-fluid">
            <img v-else src="@/assets/images/WinstonIcon512.png" class="img-fluid">
          </a>
          <div class="header_name">
            <h1 v-if="!header">Hi! Welcome to our <a id="chatbot-link" href="http://ToysForPets.com" target="_blank">ToysForPets.com</a> ShopperAssist<sup>TM</sup> Product Finder</h1>
            <h1 v-else class="" v-html="header"></h1>
          </div>
        </div>
        <div @click="startOver()" class="header_btn d-flex" style="gap:4px;">
          <span class="chat-box-reload">
            <svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1" width="10" height="10" viewBox="0 0 256 256" xml:space="preserve">

            <defs>
            </defs>
            <g style="stroke: none; stroke-width: 0; stroke-dasharray: none; stroke-linecap: butt; stroke-linejoin: miter; stroke-miterlimit: 10; fill: none; fill-rule: nonzero; opacity: 1;" transform="translate(1.4065934065934016 1.4065934065934016) scale(2.81 2.81)">
              <path d="M 75.702 53.014 c -2.142 7.995 -7.27 14.678 -14.439 18.816 c -7.168 4.138 -15.519 5.239 -23.514 3.095 c -16.505 -4.423 -26.335 -21.448 -21.913 -37.953 C 20.258 20.467 37.286 10.64 53.79 15.06 c 4.213 1.129 8.076 3.118 11.413 5.809 l -8.349 8.35 h 26.654 V 2.565 l -8.354 8.354 c -5.1 -4.405 -11.133 -7.61 -17.74 -9.381 C 33.451 -4.882 8.735 9.389 2.314 33.35 c -6.42 23.961 7.851 48.678 31.811 55.098 C 38.001 89.486 41.934 90 45.842 90 c 7.795 0 15.488 -2.044 22.42 -6.046 c 10.407 -6.008 17.851 -15.709 20.962 -27.317 L 75.702 53.014 z" style="stroke: none; stroke-width: 1; stroke-dasharray: none; stroke-linecap: butt; stroke-linejoin: miter; stroke-miterlimit: 10; fill: rgb(0,0,0); fill-rule: nonzero; opacity: 1;" transform=" matrix(1 0 0 1 0 0) " stroke-linecap="round"/>
            </g>
            </svg>
          </span>
        </div>
        <div class="header_btn d-flex" style="gap:4px;">
          <span class="chat-box-toggle">
            <svg xmlns="http://www.w3.org/2000/svg" width="10" height="10" viewBox="0 0 10 10" fill="none">
              <path d="M8.4713 1.07407L0.890182 8.65518" stroke="black" stroke-width="1.5" stroke-linecap="round"
                stroke-linejoin="round" />
              <path d="M0.890182 1.07407L8.47129 8.65518" stroke="black" stroke-width="1.5" stroke-linecap="round"
                stroke-linejoin="round" />
            </svg>
          </span>
        </div>
        
      </div>
      <div class="chat-box-body">
        <div class="chat-logs">

          <div class="chat-message my-3" :key="INDEX" v-for="(message, INDEX) in Messages">

            <div class="chat-msg user" v-if="message.content && message.role === 'user'">
              <div class="cm-msg-text">
                <h3 v-html="message.content"></h3>
              </div>
              <p v-html="message.time"></p>
            </div>

            <div
              class="chat-msg system"
              v-else-if="message.content && message.role === 'system'"
            >
              <div class="cm-answer">
                <div class="cm-msg-text" v-html="message.content"></div>
              </div>
              <div class="chat-msg chat-user">
                <p v-html="message.time"></p>
              </div>
            </div>
            <div
              class="chat-msg system"
              v-else-if="message.content && message.role === 'manual'"
            >
              <div class="cm-answer">
                <div class="cm-msg-text" v-html="message.content"></div>
              </div>
              <div class="chat-msg chat-user">
                <p v-html="message.time"></p>
              </div>
            </div>

            <div
              :class="message.content?.length?'white-swiper':'chat-msg system'"
              v-else-if="message.content && message.role === 'toys_search'"
            >
              <swiper
                v-if="message.content?.length"
                :slidesPerView="'auto'"
                :spaceBetween="10"
                :modules="modules"
                :navigation="{
                  nextEl: '.swiper-button-next',
                  prevEl: '.swiper-button-prev',
                }"
              >
                <swiper-slide :key="index" v-for="(toy,index) in message.content">
                  <div class="pet_outer mx-2">
                    <div class="pet_place_name">
                    <div v-if="toy.images.length" class="pet_images">
                    <img :src="toy.images[0].src" class="img-fluid"/>
                    </div>
                    <h5 v-html="toy.name.toString().replace(',','\n').replace('/','\n')">
                    </h5>
                    <!-- <h3>Brand : {{ toy.brand }}</h3> -->
                    <h3>Price : <span v-html="toy.price_html"></span></h3>
                    <div @click="getProduct(toy)" class="view_more text-center">
                      <div class="more_btn">More Info</div>
                    </div>
                    </div>
                  </div>
                </swiper-slide>
                <div>
                  <button class="hidebuttons swiper-button-next"></button>
                  <button class="hidebuttons swiper-button-prev"></button>
                </div>
              </swiper>
            </div>


            <div
              v-else-if="message.content && message.role === 'toy'"
            >
              <div class="pet_outer mx-2">
                <div class="pet_place_name">
                <div class="pet_images" v-if="message.content.images.length">
                 <img :src="message.content.images[0].src" class="img-fluid"/>
                </div>
                 <h6 v-html="message.content.name.toString().replace(',','\n').replace('/','\n')">
                 </h6>
                 <!-- <h6>
                    {{ message.content.description }}
                 </h6> -->
                 <!-- <h6> {{ message.content.type }} </h6> -->
                 <!-- <h6> {{ message.content.tags }} </h6> -->
                 <!-- <h3>Brand : {{ message.content.brand }}</h3> -->
                 <h3>Price : <span v-html="message.content.price_html"></span></h3>
                 <div class="view_more text-center">
                   <a class="more_btn" :href="message.content.external_url||message.content.permalink" target="_blank" >Buy Now</a>
                 </div>
                </div>
              </div>
            </div>

            <div
              v-else-if="message.content && message.role === 'default'"
            >
              <div class="pet_outer mx-2">
                <div class="local-div chat-box-btn-con">
                  <div class="chat-box-btns" @click='Ask("Toys for Dogs")'>Toys for Dogs</div>
                  <div class="chat-box-btns" @click='Ask("Toys for Cats")'>Toys for Cats</div>
                  <div class="chat-box-btns" @click='Ask("Toys for Rabbit")'>Toys for Rabbit</div>
                </div>
              </div>
            </div>

          </div>


        </div>


        <!--chat-log -->
      </div>
      <div class="chat-input">
        <div v-if="loading" class="loader_overlay">
          <div class="chat-loader"><div class="loader__element"></div></div>
        </div>
        <form @submit.prevent="askQuestion">
          <input type="text" autocomplete="off" ref="myinput" id="chat-input" v-model="question" :disabled="loading" placeholder="Write your message" />
          <button @click="askQuestion" type="submit" class="chat-submit" id="chat-submit">
            <div class="send_btn">
              <img v-if="sendImg" :src="sendImg" alt="button">
              <img v-else src="@/assets/images/Button.png" alt="button">
            </div>
          </button>
        </form>
      </div>
    </div>
  </div>
</template>

<style>
body{
  --color:#53D408;
  --cm-bot-image:url(@/assets/images/WinstonIcon512.png);
}
</style>
<style src="./bootstrap.min.css" scoped></style>
<style src="./chatbot.css" scoped></style>
