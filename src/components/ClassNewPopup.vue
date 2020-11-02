<template>
  <v-dialog v-model="isAddCommunityPopupOpen" max-width="800" persistent>
    <template v-slot:activator="{ on }">   
      <!-- <v-list-item @click="isAddCommunityPopupOpen = true">
          <v-icon left color="purple">mdi-plus</v-icon> Add class
        </v-list-item> -->
      <v-btn class="mr-2" icon v-on="on">
        <v-icon color="secondary">mdi-plus</v-icon>
      </v-btn>
    </template>
    <v-card>
      <v-card-title class="headline">Communities</v-card-title>
      <v-card-text>
        <v-row>
          <v-col cols="3" class="pr-0">
            <v-subheader class="px-0">Join an existing community</v-subheader>
          </v-col>
          <v-col cols="7" class="pa-0 d-flex align-center"> 
            <TheSearchBar 
              :items="mitClasses"
              @submit="newVal => join({ mitClass: newVal })" 
              color="accent"
            />
          </v-col>
          <v-col cols="2">
            <v-btn @click="isAddCommunityPopupOpen = false" color="secondary" text>DONE</v-btn>
          </v-col>
        </v-row>
        
        <v-row>
          <v-col cols="3" class="pr-0">
            <v-subheader class="px-0">Or create a new community</v-subheader>
          </v-col>
          <v-col cols="7">
            <v-text-field v-model="nameOfNewCommunity" label="Name" placeholder="e.g. TSR^2, GIRs, 6.036"/>
            <v-text-field v-model="descriptionOfNewCommunity" label="Description" placeholder="e.g. Intro to Machine Learning"/>
          </v-col>
          <v-col cols="2">
            <v-btn @click="createNewClass()" text color="secondary">CREATE</v-btn>
          </v-col>
        </v-row>
      </v-card-text>
    </v-card>
  </v-dialog>
</template>

<script>
import DatabaseHelpersMixin from "@/mixins/DatabaseHelpersMixin.js";
import TheSearchBar from "@/components/TheSearchBar.vue";
import db from "@/database.js";
import firebase from "firebase/app";

export default {
  mixins: [
    DatabaseHelpersMixin
  ],
  components: {
    TheSearchBar
  },
  data () {
    return {
      mitClasses: [],
      isAddCommunityPopupOpen: false,
      nameOfNewCommunity: "",
      descriptionOfNewCommunity: ""
    }
  },
  computed: {
    userRef () {
      return db.doc(`/users/${this.$store.state.user.uid}`); 
    }
  },
  async created () {
    this.mitClasses = await this.$_getCollection(db.collection("classes")); 
  },
  methods: {
    async join ({ mitClass }) {    
      this.userRef.update({
        enrolledClasses: firebase.firestore.FieldValue.arrayUnion(mitClass),
        mostRecentClassID: mitClass.id
      });
      this.$root.$emit("show-snackbar", `Successfully joined ${mitClass.name}.`);
    },
    async remove ({ mitClass }) {
      this.userRef.update({
        enrolledClasses: firebase.firestore.FieldValue.arrayRemove(mitClass)
      });
    },
    async signOut () {
      await firebase.auth().signOut(); // will trigger `onAuthStateChanged` in router.js
      this.$router.push("/");
    },
    // fundamentally create a new class, by the way that could be why there is no growth, because
    // there is no ability for people to open new classes and lounges
    async createNewClass () {
      if (!this.nameOfNewCommunity || !this.descriptionOfNewCommunity) {
        this.$root.$emit("show-snackbar", "You must enter both a name and a description.");
        return;
      }
      for (const c of this.mitClasses) {
        if (c.name === this.nameOfNewCommunity) {
          this.$root.$emit("show-snackbar", `Can't create ${this.nameOfNewCommunity} because it already exists.`)
          return; 
        }
      }

      // TODO: parallelize with Promise.all() or use getRandomId(); 
      const classDoc = await db.collection("classes").add({
        name: this.nameOfNewCommunity,
        description: this.descriptionOfNewCommunity,
        roomTypes: ["Blackboard Rooms"]
      });

      await this.userRef.update({
        enrolledClasses: firebase.firestore.FieldValue.arrayUnion({
          id: classDoc.id,
          name: this.nameOfNewCommunity,
          description: this.descriptionOfNewCommunity
        })
      });

      // now update again
      this.mitClasses = await this.$_getCollection(db.collection("classes")); 

      this.$root.$emit("show-snackbar", `${this.nameOfNewCommunity} has been created.`)
      this.nameOfNewCommunity = "";
      this.descriptionOfNewCommunity = ""; 
    }
  }
};
</script>