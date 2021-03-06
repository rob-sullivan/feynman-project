<template>
  <div>
    <div class="center">
      <big-red-button 
        class="tooltipped" 
        @click="handleEureka()"
        data-tooltip =
        "<ul>
          <li>'If you can't explain it simply, you don't understand it.'</li>
          <li>Help others and see if you grasped the material properly.</li>
          <li>It takes practise to explain things elegantly. Do you have what it takes?</li>
        </ul>">
      </big-red-button>
    </div>
    <div v-if="questions[0]">
      <div class="container">
        <ul v-if="activeFeynmen.length != 0" class="collection with-header black">
          <li class="collection-header">
            <h4 class="black-text">Feynmen</h4>
          </li>
          <template v-for="f in activeFeynmen">
            <li :key="f.uid" class="collection-item avatar white">
              <i class="material-icons circle grey">person</i>
              <span class="title black-text">Feynman #{{ f.feynmanNumber }}</span>
              <p v-if="f.finished" class="black-text">Finished</p>
              <Promised :promise="checkOnline(f)">
                <p>Fetching online status...</p>
                <p slot-scope="data" class="green-text">{{ data }}</p>
                <p slot="catch" slot-scope="error">Error: {{ error.message }}</p>
              </Promised>
              <a @click="enterChat(f)" class="secondary-content btn-floating pulse pink">
                <i class="material-icons white-text">email</i>
              </a>
            </li>
          </template>
        </ul>
      </div>
      <template v-for="(group, idx) in chainReactionGroups">
        <chain-reaction :list-of-nodes="getParticipants(group)" :groupNumber="idx" :key="idx"></chain-reaction>
      </template>
    </div>
  </div>
</template>

<script>
import firebase from 'firebase'
import Promised from 'vue-promised'
import ChainReaction from './ChainReaction.vue'
import BigRedButton from './BigRedButton.vue'
import db from '@/firebase/init.js'

export default {
  components: {
    ChainReaction,
    Promised,
    BigRedButton
  },
  data () {
    return {
      loading: true,
      questions: []
    }
  },
  mounted () {
    this.$bind('questions', db.collection('questions').where('questionID', '==', this.$route.path))
    .then(doc => {
      this.loading = false
    })
    .catch(error => {
      console.log('error from connecting to Firestore: ', error)
    })
  },
  computed: {
    user () {
      return this.$store.state.user
    },
    chainReactionGroups () {
      var groups = [] 
      this.questions[0].feynmen.forEach(f => {
        if (f.chainReactionCreatorUID) {
          if (!groups.includes(f.chainReactionCreatorUID)) {
            groups.push(f.chainReactionCreatorUID)
          }
        }
      })
      return groups 
    },
    activeFeynmen () {
      if (!this.questions[0]) {
        return []
      } else if (this.questions[0]) {
        var output = this.questions[0].feynmen 
        output = output.filter(f => f.chainReactionCreatorUID != null)
        output = output.filter(f => f.retired != true || f.retired == null)
        return output 
      }
    }
  },
  methods: {
    getParticipants (group) {
      return this.questions[0].feynmen.filter(f => f.chainReactionCreatorUID == group)
    },
    async enterChat ({ uid, finished, displayName, chainReactionCreatorUID }) {
      // cannot chat with yourself 
      if (this.user.uid == uid) {
        return 
      }
      // if you're asking a Feynman, record it 
      if (chainReactionCreatorUID) {
        this.questions[0].feynmen.forEach(f => {
          if (f.uid == this.user.uid) {
            // don't fuck with anything if the user is already part of a reaction
            if (f.chainReactionCreatorUID) {
              console.log('user is already part of a chain reaction')
              return 
            } else {
              const currentFeynman = {
                displayName,
                uid,
                chainReactionCreatorUID
              }
              f.mostRecentFeynman = currentFeynman
            }
          }
        })
        const ref = db.collection('questions').doc(this.questions[0].id)
        const updatedFeynmen = this.questions[0].feynmen 
        await ref.update({
          feynmen: updatedFeynmen
        })
      }
      // create a chat room 
      const sortedUIDs = [this.user.uid, uid].sort()
      const roomId = sortedUIDs.join('') // ensure order of UIDs stays consistent no matter who asks
      const doc = db.collection('chatRooms').doc(roomId)
      const chatRoom = await doc.get()
      const currentUser = {
        displayName: this.user.displayName,
        uid: this.user.uid 
      }
      const feynman = {
        displayName,
        uid
      }
      // ideally, create the chatroom only in one place 
      if (!chatRoom.data()) {
        await doc.set({
          messages: [],
          participants: [currentUser, feynman],
        })
        const whiteboardRef = db.collection('whiteboards').doc(roomId) 
        await whiteboardRef.set({
          allPaths: []
        })
        console.log('successfully created whiteboard document')
      }
      this.$router.push('/chat/' + roomId)
      // TODO: notify user 
    },
    async checkOnline ({ uid }) {
      const ref = db.collection('users').doc(uid)
      const feynman = await ref.get()
      if (feynman.exists) {
        const result = feynman.data().isOnline
        if (!result || result == false) {
          return 'offline'
        } else {
          return 'online'
        } 
      }
    },
    async handleEureka () {
      var joinedReaction = false 
      this.questions[0].feynmen.forEach(f => {
        if (f.uid == this.user.uid && f.mostRecentFeynman) {
          console.log('user had a teacher - adding him/her to a chain reaction')
          f.teacher = f.mostRecentFeynman.uid
          f.chainReactionCreatorUID = f.mostRecentFeynman.chainReactionCreatorUID
          joinedReaction = true 
        }
      })
      const updatedFeynmen = this.questions[0].feynmen 
      // user was not taught by anyone - let him start his own chain reaction
      if (!joinedReaction) {
        console.log('student was alone - starting a new chain reaction')
        this.questions[0].feynmen.forEach(f => {
          if (f.uid == this.user.uid) {
            f.teacher = "Richard Feynman's UID"
            f.chainReactionCreatorUID = f.uid 
          }
        })
      }
      const ref = db.collection('questions').doc(this.questions[0].id)
      await ref.update({
        feynmen: updatedFeynmen
      })
    }
  }
}
</script>
