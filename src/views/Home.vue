<template>
  <div class="home">
    <!--<img alt="Vue logo" src="../assets/logo.png">-->
    <!--<HelloWorld msg="Welcome to Your Vue.js App"/>-->
      <div style="margin-bottom: 50px" v-if="!fetchedRoleId">
          token id: <input v-model="roleId" type="number">
          <button class="btn btn-dark" @click="createRole" :disabled="fetchingRoleId" style="margin-left: 20px" >Create Role</button>
      </div>
      <div class="grid-container">
          <ERC998 class="grid-item" chain="public"  :event="pbcEvent" :ws="ws" :avatar=avatar :reqCode="reqCode"> </ERC998>
          <ERC998 class="grid-item" chain="private" :event="pvcEvent" :ws="ws" :avatar=avatar :reqCode="reqCode"> </ERC998>
      </div>
  </div>
</template>

<script>
// @ is an alias to /src
// const HelloWorld = () => import('@/components/HelloWorld.vue')

export default {
    name: 'home',
    components: {
        ERC998: () => import('@/components/ERC998.vue')
    },
    data() {
        return {
            avatar: {
                role: ()=>import('@/assets/avatarPics/role.jpg'),
                weapon: ()=>import('@/assets/avatarPics/weapon.jpg'),
                armor: ()=>import('@/assets/avatarPics/armor.jpg'),
                roleLocked: ()=>import('@/assets/avatarPics/roleLocked.jpg'),
                weaponLocked: ()=>import('@/assets/avatarPics/weaponLocked.jpg'),
                armorLocked: ()=>import('@/assets/avatarPics/armorLocked.jpg'),
                // lock: ()=>import('../assets/avatarPics/lock.jpg')
            },
            roleId: undefined,
            fetchedRoleId: false,
            fetchingRoleId:false,
            ws:undefined,
            pbcEvent: undefined,
            pvcEvent: undefined,
            reqCode :{
                createERC998: 1,
                createERC721: 2,
                transferERC998: 3,
                transferERC721: 4,
            }
        }
    },
    methods: {
      createRole: function() {
          let payload = {
              "gcuid": 1,
              "account": "0x3c62aa7913bc303ee4b9c07df87b556b6770e3fc",
              "tokenId": eval(this.roleId),
              "uri": "secdev/"+ eval(this.roleId)
          }
          this.fetchingRoleId = true
          this.ws.send(JSON.stringify(payload))
      }
    },
    created: function(){
        let config = require('@/etc/config.json')
        this.ws = new WebSocket(config.serverPath)
        this.ws.onopen = e => {
            console.log("websocket open")
        }
        this.ws.onmessage = e => {
            let res = JSON.parse(e.data)
            switch (res.gcuid) {
                case this.reqCode.createERC998:
                    if (res.status) {
                        //disable button
                        //TODO
                        this.fetchedRoleId = true
                        this.pvcEvent = res
                    } else {
                        console.log(res.reason)
                    }
                    this.fetchingRoleId = false
                    break
                case this.reqCode.createERC721:
                    if (res.status) {
                        this.pvcEvent = res
                    } else {
                        console.log(res.reason)
                    }
                    break
                case this.reqCode.transferERC998:
                    if (res.status) {
                        this.pvcEvent = res
                        this.pbcEvent = res
                    } else {
                        console.log(res.reason)
                    }
                    break
                case this.reqCode.transferERC721:
                    if (res.status) {
                        this.pvcEvent = res
                        this.pbcEvent = res
                    } else {
                        console.log(res.reason)
                    }
                    break
                default:
                    console.log("unknown response")
            }
        }
        this.ws.onclose = e=> {
            console.log("websocket close")
        }
    }
}
</script>
