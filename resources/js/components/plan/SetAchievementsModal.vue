<template>
    <modal
        id="set-achievements-modal"
        name="set-achievements-modal"
        height="auto"
        :adaptive=true
        draggable=".draggable"
        @before-open="beforeOpen"
        @opened="opened()"
        @before-close="beforeClose()"
        style="z-index: 1100"
    >
        <div class="card mb-0" style="max-height: 100svh;">
            <div class="card-header">
                <h3 class="card-title">{{ trans('global.plan.select_users') }}</h3>
                <div class="card-tools">
                    <button type="button" class="btn btn-tool draggable">
                        <i class="fa fa-arrows-alt"></i>
                    </button>
                    <button type="button" class="btn btn-tool" data-widget="remove" @click="close()">
                        <i class="fa fa-times"></i>
                    </button>
                </div>
            </div>
            <div class="card-body overflow-auto">
                <table
                    id="achievements-table"
                    class="table m-0 border-top-0 dataTable"
                    v-permission="'achievement_access'"
                >
                    <thead>
                        <tr class="border-top-0">
                            <th style="width: 0px;"></th>
                            <th class="sorting sorting_asc" @click="sortBy(1)">{{ trans('global.firstname') }}</th>
                            <th class="sorting" @click="sortBy(2)">{{ trans('global.lastname') }}</th>
                            <th>{{ trans('global.notes') }}</th>
                            <th class="sorting" @click="sortBy(4)">Status</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr class="thick-line">
                            <td>
                                <input
                                    type="checkbox"
                                    v-model="checkAll"
                                    @change="toggleUsers()"
                                />
                            </td>
                            <td colspan="2">
                                {{ selectedUsers.length  + ' ' + trans('global.users_selected')}}
                            </td>
                            <td>
                                <i
                                    class="far fa-sticky-note text-muted p-1"
                                    :class="selectedUsers.length === 0 ? 'text-gray' : 'pointer'"
                                    style="font-size: 18px; margin: -0.25rem"
                                    @click.prevent="openSelectedNotes()"
                                ></i>
                            </td>
                            <td>
                                <AchievementIndicator
                                    class="mr-3"
                                    v-permission="'achievement_create'"
                                    :objective="objective.default"
                                    :type="'enabling'"
                                    :users="selectedUsers"
                                    :settings="{'achievements' : false, 'edit': false, 'sendStatus': true}"
                                    :disabled="selectedUsers.length === 0"
                                ></AchievementIndicator>
                            </td>
                        </tr>
                        <tr v-for="user in users">
                            <td>
                                <input
                                    type="checkbox"
                                    :value="user.id"
                                    v-model="selectedUsers"
                                />
                            </td>
                            <td>{{ user.firstname }}</td>
                            <td>{{ user.lastname }}</td>
                            <td>
                                <i style="font-size: 18px; margin: -0.25rem" 
                                    class="far fa-sticky-note text-muted pointer p-1"
                                    @click.prevent="openNotes(user.id)"
                                ></i>
                            </td>
                            <td :data-value="objective[user.id]?.achievements[0].status ?? '00'">
                                <AchievementIndicator
                                    v-permission="'achievement_create'"
                                    :objective="objective[user.id]"
                                    :type="'enabling'"
                                    :users="[user.id]"
                                    :settings="{'achievements' : false, 'edit': false}"
                                ></AchievementIndicator>
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>
            <div class="card-footer">
                <span class="d-flex justify-content-between">
                    <button type="button" class="btn btn-default" data-widget="remove" @click="close()">{{ trans('global.close') }}</button>
                    <button class="btn btn-primary" @click="submit()">{{ trans('global.save') }}</button>
                </span>
            </div>
        </div>
    </modal>
</template>
<script>
const AchievementIndicator = () => import('./../objectives/AchievementIndicator.vue');

export default {
    props: {
        users: [],
    },
    data() {
        return {
            objective: {},
            selectedUsers: [],
            checkAll: false,
            sortByAsc: false,
        };
    },
    mounted() {},
    methods: {
        close() {
            this.$modal.hide('set-achievements-modal');
            this.objective = {};
        },
        submit() {
            this.close();
        },
        /**
         * since we need the achievement-status combined with the specific user,
         * we'll create an object with the user_id as the attribute
         * and set a default for users with no achievement
         * INFO: I don't want to do a double for-loop because O(n^2)
         */
        beforeOpen(event) {
            const eventObj = event.params.objective;
            const obj = {}; // if we add attributes directly to 'this.objective' it won't work
            obj.default = { id: eventObj.id }; // needed for group selection
            // create an achievement-placeholder for every user
            this.users.forEach(user => {
                obj[user.id] = {
                    achievements: [{ status: '00' }],
                    id: eventObj.id
                };
            })
            // connect achievements based of its user-id
            eventObj.achievements.forEach(achievement => {
                obj[achievement.user_id].achievements[0] = achievement;
            });

            this.objective = obj;
        },
        beforeClose() {
            this.$parent.updateAchievements(this.objective.default.id);
        },
        opened() {
            this.sortByAsc = false;
            this.sortBy(1);
        },
        closed() {},
        sortBy(columnNr = 1) {
            const table = document.querySelector('#achievements-table tbody');
            const trs = table.querySelectorAll('tr:not(:nth-child(1))'); // first tr is the group-selection row

            Array.from(trs)
                .sort((a, b) => {
                    // TODO: if two rows have the same value, further sort by other column
                    switch (columnNr) {
                        // TODO: 'Status'-sort needs overhaul
                        case 4: // column 4 = 'Status'
                            return this.sortByAsc
                                ? b.cells[columnNr].getAttribute('data-value') - a.cells[columnNr].getAttribute('data-value')
                                : a.cells[columnNr].getAttribute('data-value') - b.cells[columnNr].getAttribute('data-value');
                        case 1: // column 1 = 'Firstname' => default
                        case 2: // column 2 = 'Lastname'
                        default:
                        return this.sortByAsc
                            ? a.cells[columnNr].innerText.localeCompare(b.cells[columnNr].innerText) * -1 // reverse logic
                            : a.cells[columnNr].innerText.localeCompare(b.cells[columnNr].innerText);
                    }
                }).forEach(tr => table.appendChild(tr));


            // hide previous sorting indicator
            const previousSort = this.sortByAsc ? 'sorting_desc' : 'sorting_asc';
            this.$el.querySelector('.' + previousSort).classList.remove(previousSort)
            // show current sorting indicator
            const th = document.querySelector('#achievements-table').firstChild.firstChild.children[columnNr];
            const currentSort = this.sortByAsc ? 'sorting_asc' : 'sorting_desc';
            th.classList.add(currentSort);

            this.sortByAsc = !this.sortByAsc;
        },
        /**
         * creates a new achievement with unset status </br>
         * if achievement already exists, its status will be set back to default = '00'
         * @param {Number} user_id 
         * @returns {Number} id of newly created achievement
         */
        async createAchievement(user_id) {
            const objective_id = this.objective.default.id;

            const achievement = {
                'referenceable_type': 'App\\EnablingObjective',
                'referenceable_id': objective_id,
                'user_id': user_id,
                'status': '0' // only one zero, setting '00' will throw 500
            }
            // create a new achievement-entry and get the ID
            const achievement_id = (await axios.post('/achievements', achievement)).data.id;

            // add new achievement to data, in case it is needed
            this.objective[user_id] = {
                achievements: [{ id: achievement_id, status: '00' }],
                id: objective_id,
            };

            return achievement_id;
        },
        /**
         * when opening the note-modal, an achievement-id is needed to create a note
         * if there's no achievement, create one with unset status and get a new ID
         * @param {Number} user_id 
         */
        async openNotes(user_id) {
            let achievement_id = this.objective[user_id]?.achievements[0].id; // check if achievement exists

            if (achievement_id === undefined) {
                achievement_id = await this.createAchievement(user_id);
            }

            this.$modal.show('note-modal', {
                'method': 'post',
                'notable_type': 'App\\Achievement',
                'notable_id': achievement_id,
                'show_tabs': false,
            });
        },
        /**
         * get achievement for every selected user or create one if not exists
         */
        async openSelectedNotes() {
            if (this.selectedUsers.length === 0) return;
            // gets an object-structure like this { user_id: achievement_id | undefined }
            let achievements = {};
            let users = {};

            this.selectedUsers.forEach(user_id => {
                achievements[user_id] = this.objective[user_id]?.achievements[0].id;
                // get names of each user for the name-label in notes
                const user = this.users.find(user => user.id === user_id);
                users[user_id] = `${user.firstname} ${user.lastname}`;
            });

            for (const [user_id, value] of Object.entries(achievements)) {
                if (value === undefined) {
                    // changes value from undefined to achievement_id
                    achievements[user_id] = this.createAchievement(user_id);
                }
            }

            this.$modal.show('note-modal', {
                'method': 'post',
                'notable_type': 'App\\Achievement',
                'notable_id': Object.values(achievements), // array of achievement_ids
                'show_tabs': false,
                'users': users,
            });
        },
        /**
         * function is called from AchievementIndicator to overwrite selected statuses
         * @param {string} status the second char of status
         */
        updateStatus(status) {
            this.selectedUsers.forEach(id => {
                let selfStatus = this.objective[id].achievements[0].status[0];
                // only overwrite the second char of the status
                this.objective[id].achievements[0].status = selfStatus + status;
            });
        },
        toggleUsers() {
            this.selectedUsers = this.checkAll
                ? this.users.map(user => user.id)
                : [];
        },
    },
    components: {
        AchievementIndicator
    }
}
</script>
<style scoped>
.thick-line > td {
    border-bottom: 2px solid #dee2e6;
}
</style>