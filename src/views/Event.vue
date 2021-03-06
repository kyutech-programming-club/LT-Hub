<template>
  <div class="event">
    <div v-if="event.id">
      <div>
        <h1>{{ event.title }}</h1>
        <v-container v-if="event.author.id == currentUserId">
          <v-row justify="center">
            <v-col cols="2" class="pa-0 mt-2">
              <edit-event-form
                v-if="isBeforeEvent"
                :event="event"/>
            </v-col>
            <v-col cols="2" class="pa-0 mt-2">
              <v-chip
                class="ma-2"
                color="red"
                text-color="white"
                @click="deleteEvent">
                <v-icon>
                  mdi-delete
                </v-icon>
              </v-chip>
            </v-col>
          </v-row>
        </v-container>
        <div v-if="event.start">
          期間：{{ getStringFromDate(this.event.start.toDate()).substr(0,16) }} ~ {{ getStringFromDate(this.event.end.toDate()).substr(0,16) }}<br>
        </div>
        概要
        <v-card
          v-scroll.self="onScroll"
          class="overflow-y-auto reline"
          max-height="400"
          flat>
          {{event.description}}<br>
        </v-card>
        場所：
        <template v-for="(content, id) in splitComment(event.place)">
          <span :key="id" v-if="validUrl(content)" class="text-left reline"><a :href="content" target="_blank" rel="noopener noreferrer">{{content}}</a></span>
          <span :key="id" v-else class="text-left reline">{{content}}</span>
        </template><br>
        <div v-if="event.createdTime">
          作成日時：{{ getStringFromDate(event.createdTime.toDate()) }}<br>
        </div>
        <div v-if="event.updatedTime">
          最終更新日時：{{ getStringFromDate(event.updatedTime.toDate()) }}
        </div>
      </div>
      <div v-if="event.author.id">
        責任者：
        <user-item-small
          :user = "event.author" />
      </div>
    </div>
    <div v-if="isBeforeEvent && currentUserId">
      <div  v-if="isParticipated">
        <v-container>
          <v-row dense justify="center" tag="span">
            <v-col class="mt-1">
              <v-btn class="white--text font-weight-bold" color="#ff4b4b" @click="cancelParticipate">
                Cancel
              </v-btn>
            </v-col>
            <v-col>
              <new-talk-form
                :eventId="event.id"
                :userId="currentUserId"/>
            </v-col>
          </v-row>
        </v-container>
      </div>
      <div v-else>
        <v-btn
          class="white--text font-weight-bold"
          color="#009eff"
          @click="participate">
          Join
        </v-btn>
      </div>
    </div>
    <div class="talks-list" v-if="event.order && event.author">
      LT数 {{event.order.length}}
      <div v-if="event.author.id == currentUserId">
        <v-chip
          v-if="isBeforeEvent"
          class="ma-2"
          color="green"
          text-color="white"
          @click="saveEventOrder">
          <v-icon left>
            mdi-account-switch
          </v-icon>
          Save order
        </v-chip>
      </div>
      <draggable
        v-if="event.author.id === currentUserId"
        :list="orderItem"
      >
        <talk-item
          v-for="talkId in orderItem"
          :key="talkId"
          :talkId="talkId" />
      </draggable>
      <talk-item
        v-else
        v-for="talkId in orderItem"
        :key="talkId"
        :talkId="talkId"
      />
    </div>
    <div class="talks-list" v-else>
      LT数 {{talks.length}}
      <talk-item
        v-for="talk in talks"
        :key="talk.id"
        :talkId="talk.id"/>
    </div>

    <div v-if="participants" class="users-list">
      参加者数  {{participants.length}}人<br />
      <participate-item
        v-for="user in participants"
        :key="user.id"
        :user="user" />
    </div>
  </div>
</template>

<script>
  import firebase from 'firebase'
  import EditEventForm from '@/components/EditEventForm.vue'
  import NewTalkForm from '@/components/NewTalkForm.vue'
  import ParticipateItem from '@/components/ParticipateItem.vue'
  import UserItemSmall from '@/components/UserItemSmall.vue'
  import TalkItem from '@/components/TalkItem.vue'
  import { db } from '@/firebase/firestore.js'
  import draggable from 'vuedraggable';

  export default {
    name: 'Event',
    components: {
      ParticipateItem,
      EditEventForm,
      UserItemSmall,
      TalkItem,
      NewTalkForm,
      draggable,
    },
    data() {
      return {
        event: [],
        participants: [],
        isParticipated: false,
        author: {},
        currentUserId: '',
        participated: false,
        isAuthor: false,
        isBeforeEvent: false,
        orderItem: [],
        talks: []
      }
    },
    created() {
      console.log("created");
      let self = this;
      firebase.auth().onAuthStateChanged(async(user) => {
        if (user != null) {
          this.$root.$set(self, 'currentUserId', user.uid);
        }
      });
    },
    watch: {
      async event() {
        if (this.currentUserId != '') {
          let currentUserRef = await db.collection('users').doc(this.currentUserId)
          let currentEventRef = await db.collection('events').doc(this.$route.params['id'])
          let participantRef = await db.collection('participants')
            .where('userRef', '==', currentUserRef)
            .where('eventRef', '==', currentEventRef)
            .get();
          if (!participantRef.empty) {
            this.isParticipated = true;
          }
        }
        let now = new Date();
        if (this.event.end.toDate() > now) {
          this.isBeforeEvent = true;
        }

        if (this.event.order) {
          this.orderItem = await this.event.order;
        }
      },
    },
    firestore(){
      console.log("firestore");
      return {
        event: db.collection('events').doc(this.$route.params['id']),
        participants: db.collection('participants')
          .where('eventRef', '==', db.collection('events').doc(this.$route.params['id'])),
        talks: db.collection('talks')
          .where('eventRef', '==', db.collection('events').doc(this.$route.params['id'])),
      }
    },
    methods: {
      async deleteEvent() {
        var res = confirm('ほんとにイベントを取りやめますか？？？？？');
        if (res) {
          console.log('deleteEvent');
          let eventRef = await db.collection('events').doc(this.event.id); //参加イベントの参照オブジェクト
          db.collection('participants').where('eventRef', '==', eventRef)
            .get()
            .then(participants =>{
              participants.forEach(participant => {
                participant.ref.delete();
              })
            });

          db.collection('talks').where('eventRef', '==', eventRef)
            .get()
            .then(talks =>{
              talks.forEach(talk => {
                talk.ref.delete();
              })
            });

          eventRef
            .delete()
            .then(() => {
              this.$router.push({name: 'events'});
            })
            .catch(err => {
              console.error('Error deleting event data: ', err);
            });
        }
      },
      async participate() {
        let self = this;
        let userRef = await db.collection('users').doc(self.currentUserId); //ログインユーザーの参照オブジェクト
        let eventRef = await db.collection('events').doc(self.event.id); //参加イベントの参照オブジェクト
        db.collection('participants').add({
          userRef: userRef,
          eventRef: eventRef
        });
        this.isParticipated = true;
      },
      async cancelParticipate() {
        var res = confirm('参加を取り消しますか？');
        if (res) {
          this.isParticipated = false;
          let currentUserRef = await db.collection('users').doc(this.currentUserId)
          let currentEventRef = await db.collection('events').doc(this.$route.params['id'])
          let talkIds = [];
          db.collection('participants')
            .where('userRef', '==', currentUserRef)
            .where('eventRef', '==', currentEventRef)
            .get()
            .then(participants => {
              participants.forEach(participant =>{
                participant.ref.delete();
              })
            });
          await db.collection('talks')
            .where('userRef', '==', currentUserRef)
            .where('eventRef', '==', currentEventRef)
            .get()
            .then(talks => {
              talks.forEach(talk =>{
                talk.ref.delete();
                talkIds.push(talk.ref.id);
              })
            });
          currentEventRef.update({
            order: firebase.firestore.FieldValue.arrayRemove(...talkIds),
          });
          alert('ぴえん');
        } else {
          alert('命拾いしましたね');
        }
      },
      //日付から文字列に変換する関数
      getStringFromDate(date) {
        var year_str = date.getFullYear();
        //月だけ+1すること
        var month_str = 1 + date.getMonth();
        var day_str = date.getDate();
        var hour_str = date.getHours();
        var minute_str = date.getMinutes();
        var second_str = date.getSeconds();

        month_str = ('0' + month_str).slice(-2);
        day_str = ('0' + day_str).slice(-2);
        hour_str = ('0' + hour_str).slice(-2);
        minute_str = ('0' + minute_str).slice(-2);
        second_str = ('0' + second_str).slice(-2);

        var format_str = 'YYYY/MM/DD hh:mm:ss';
        format_str = format_str.replace(/YYYY/g, year_str);
        format_str = format_str.replace(/MM/g, month_str);
        format_str = format_str.replace(/DD/g, day_str);
        format_str = format_str.replace(/hh/g, hour_str);
        format_str = format_str.replace(/mm/g, minute_str);
        format_str = format_str.replace(/ss/g, second_str);

        return format_str;
      },
      onScroll () {
        this.scrollInvoked++
      },
      saveEventOrder() {
        console.log(this.orderItem)
        db.collection('events').doc(this.event.id).
        update({
          order: this.orderItem
        })
      },
      splitComment: function(comment) {
        return comment.toString().split(/(https?:\/\/[-_.!~*'()a-zA-Z0-9;/?:@&=+$,%#]+)/g);
      },
      validUrl(checkText){
        return checkText.match(/^(https?|ftp)(:\/\/[-_.!~*'()a-zA-Z0-9;/?:@&=+$,%#]+)$/);
      },
    }
  }
</script>

<style scoped>
  .reline {
    white-space: pre-wrap;
    word-wrap: break-word;
  }
</style>
