<template>
    <div class="container">
        <h1>{{chain}} chain</h1>
        <div >
            <ul style="margin-top: 40px" v-for="k in displayedAvatarProperty">
                <span  v-if="isObjectExist(k)">
                    <img :src="getObject(k)" :class="k" />
                    <button v-if="!isObjectLocked(k)" class="btn btn-dark" style="margin-left: 20px" :disabled="disableRules('transfer',k)" @click="transfer(k)">{{`transfer ${k}`}}</button>
                </span>
                <span v-else-if="isObjectExist('role')">
                    <span v-if="equipmentNotExist(k)">
                        <div style="margin-bottom: 50px">
                            {{k}} id: <input v-model="formInput[k]" type="number">
                            <button class="btn btn-dark" style="margin-left: 20px" :disabled="disableRules('create',k)" @click="create(k)">{{`create ${k}`}}</button>
                        </div>
                    </span>
                </span>
            </ul>
        </div>
    </div>
</template>

<script>
    export default {
        name: 'ERC998',
        props: {
            chain: String,
            avatar: Object,
            event: Object,
            ws: WebSocket,
            reqCode: Object,
        },
        data: function () {
            return {
                weapon: undefined,
                armor: undefined,
                role: undefined,
                onlyPrivate: true,  //used for first create in private chain disable other transfer button
                avatarStatus: {
                    roleStatus: 0,
                    weaponStatus: 0,
                    armorStatus: 0,
                },
                statusCode: {
                    unknown: 0,
                    lock: 1,
                    unlock: 2,
                },
                formInput: {
                    armor: undefined,
                    weapon: undefined,
                    role: undefined
                },
                roleId: undefined,
                weaponId: undefined,
                armorId: undefined,
                displayedAvatarProperty: ["role","weapon","armor"],
                buttonStatus: {
                    transfer: {
                        role:false,
                        weapon:false,
                        armor:false
                    },
                    create: {
                        role:false,
                        weapon:false,
                        armor: false
                    }
                }
            }
        },
        methods: {
            isObjectLocked: function(objectName) {
                return this.avatarStatus[objectName] === this.statusCode.lock
            },
            disableRules: function(action, buttonName) {
                let tag =  this.buttonStatus[action][buttonName]
                if (this.chain === "private" && action === "transfer" && buttonName !== "role") {
                    tag = tag || this.onlyPrivate
                }
                return tag
            },
            equipmentNotExist: function(objectName) {
                return this[objectName+"Id"] === undefined && this.chain==="private"
            },
            getObject: function(objectName){
                return this[objectName]
            },
            isObjectExist: function(objectName) {
                return this[objectName] !== undefined
            },
            transfer: function (objectName) {
                this.buttonStatus["transfer"][objectName] = true
                let payload
                switch (objectName) {
                    case("role"):
                        console.log("roleID"+this.roleId)
                        payload = {
                            "gcuid": this.reqCode.transferERC998,
                            "chain": this.chain,
                            "tokenId": eval(this.roleId),
                        }
                        break
                    case("weapon"):
                    case("armor"):
                        let childTokenId = eval(this[objectName+"Id"])
                        console.log(objectName+"  id: "+ childTokenId)
                        payload = {
                            "gcuid": this.reqCode.transferERC721,
                            "chain": this.chain,
                            "object": objectName,
                            "parentTokenId": this.roleId,
                            "childTokenId": childTokenId,
                            "uri": "secdev/"+ childTokenId,
                        }
                        break
                    default:
                        console.log("no such role");
                        return
                }
                this.ws.send(JSON.stringify(payload))
            },
            create: function(objectName) {
                this.buttonStatus["create"][objectName] = true
                let payload = {
                    "gcuid": this.reqCode.createERC721,
                    "object": objectName,
                    "parentTokenId": eval(this.roleId),
                    "childTokenId": eval(this.formInput[objectName]),
                    "uri":"secdev/"+ eval(this.formInput[objectName]),
                }
                this.ws.send(JSON.stringify(payload))
            },
            setObjectsPic: function(...objectNames){
                objectNames.forEach(objectName=>{
                    this.avatar[objectName]().then((path=>{
                        this[objectName] = path.default
                    }))
                })
            },
            setObjectPic: function(object, pic) {
                this.avatar[pic]().then(path=>{
                    this[object] = path.default
                })
            },
            deleteObjectsPic: function(...objectNames) {
                objectNames.forEach(objectName => {
                    this[objectName] = undefined
                })
            },
            switchRoleStatus: function(roleStatus) {
                let role = "role"
                if(roleStatus == this.statusCode.lock) {
                    this.setObjectPic(role, role+"Locked")
                } else if (roleStatus == this.statusCode.unlock){
                    this.setObjectsPic(role)
                } else if (roleStatus == this.statusCode.unknown) {
                    this.deleteObjectsPic(role)
                } else {
                    console.log("role status unknown")
                }
            },
            switchEquipmentStatus: function(equipmentName, equipMentStatus) {
                console.log(equipMentStatus, equipmentName)
                if(equipMentStatus == this.statusCode.unlock) {
                    this.setObjectsPic(equipmentName)
                } else if (equipMentStatus == this.statusCode.lock){
                    this.setObjectsPic(equipmentName+"Locked")
                    this.setObjectPic(equipmentName, equipmentName+"Locked")
                } else if (equipMentStatus == this.statusCode.unknown){
                    this.deleteObjectsPic(equipmentName)
                } else {
                    console.log("unknown equiment status code")
                }
            },
        },
        created: function () {
        },
        watch: {
            event: function (e) {
                switch (e.gcuid) {
                    case this.reqCode.createERC998:
                        this.avatarStatus["role"] = this.statusCode.unlock
                        this.switchRoleStatus(this.statusCode.unlock)
                        this.roleId = e.tokenId
                        break
                    case this.reqCode.createERC721:
                        this.buttonStatus["create"][e.object] = false
                        console.log("create " + e.object)
                        console.log(typeof e.object)
                        this.avatarStatus[e.object] = this.statusCode.unlock
                        this.switchEquipmentStatus(e.object, this.statusCode.unlock)
                        this[e.object+"Id"] = e.childTokenId
                        break
                    case this.reqCode.transferERC998:
                        this.onlyPrivate = false
                        this.buttonStatus["transfer"]["role"] = false
                        if(e.chain===this.chain) {
                            this.avatarStatus["role"] = this.statusCode.lock
                            this.switchRoleStatus(this.statusCode.lock)
                        } else {
                            if(this.roleId == undefined) {
                                this.roleId = e.tokenId
                            }
                            this.avatarStatus["role"] = this.statusCode.unlock
                            this.switchRoleStatus(this.statusCode.unlock)
                        }
                        break
                    case this.reqCode.transferERC721:
                        this.buttonStatus["transfer"][e.object] = false
                        if(e.chain === this.chain) {
                            this.avatarStatus[e.object] = this.statusCode.lock
                            this.switchEquipmentStatus(e.object, this.statusCode.lock)
                        } else {
                            if(this[e.object+"Id"] === undefined) {
                                this[e.object+"Id"] =  e.childTokenId
                            }
                            this.avatarStatus[e.object] = this.statusCode.unlock
                            this.switchEquipmentStatus(e.object, this.statusCode.unlock)
                        }
                        break;
                    default:
                        console.log("unknown response")
                }
            },
        }
    }
</script>

<style scoped>
</style>
