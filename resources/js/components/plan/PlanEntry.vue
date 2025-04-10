<template >
    <div class="row ">
        <div class="col-12 pt-2">
            <div class="card">
                <div v-if="create">
                    <div class="card-header pointer"
                         @click="edit()"
                    >
                        <i class="fas fa-add pr-1"></i>
                        {{ trans('global.planEntry.create') }}
                    </div>
                </div>
                <div v-else>
                    <div :id="'plan-entry-' + entry.id"
                         v-if="!editor"
                         :style="{ 'border-left-style': 'solid', 'border-radius': '0.25rem', 'border-color': entry.color }">
                        <div class="card-header collapsed" data-toggle="collapse" :data-target="'#plan-entry-' + entry.id + ' > .card-body'" aria-expanded="false">
                            <i class="mr-1"
                            :class="entry.css_icon"></i>
                            {{ entry.title }}
                            <i class="fa fa-angle-up"></i>
                            <div v-if="showTools"
                                class="card-tools">
                                <i class="fa fa-pencil-alt mr-2 pointer link-muted"
                                   @click="edit()"></i>
                                <i class="fas fa-trash pointer text-danger"
                                   @click.stop="$parent.$parent.confirmEntryDelete(entry.id)"></i>
                            </div>
                        </div>
                        <div class="card-body py-2 collapse">
                            <img v-if="Number.isInteger(entry.medium_id)"
                                 class="pull-right"
                                 :src="'/media/' + entry.medium_id + '/thumb'"/>
                            <span v-dompurify-html="entry.description ?? ''"></span>

                            <objectives
                                referenceable_type="App\PlanEntry"
                                :referenceable_id="entry.id"
                                :owner_id="entry.owner_id"
                                :editable="editable"
                                :showTools="showTools"
                            ></objectives>

                            <Trainings
                                :plan="plan"
                                subscribable_type="App\PlanEntry"
                                :subscribable_id="entry.id"
                            ></Trainings>
                        </div>
                    </div>
                </div>
                <div v-if="editor"
                     class="card-body"
                >
                    <color-picker-input
                        v-model="form.color"
                    ></color-picker-input>

                    <div class="form-group">
                        <input
                            type="text"
                            id="title"
                            name="title"
                            class="form-control"
                            v-model.trim="form.title"
                            :placeholder="trans('global.planEntry.fields.title')"
                            required
                        />
                        <p class="help-block" v-if="form.errors.title" v-text="form.errors.title[0]"></p>
                    </div>

                    <div class="form-group">
                        <textarea
                            id="description"
                            name="description"
                            :placeholder="trans('global.planEntry.fields.description')"
                            class="form-control description my-editor"
                            v-model.trim="form.description"
                        ></textarea>
                        <p class="help-block" v-if="form.errors.description" v-text="form.errors.description[0]"></p>
                    </div>
                    <div class="form-group">
                        <font-awesome-picker
                            :searchbox="trans('global.select_icon')"
                            v-on:selectIcon="setIcon"
                        ></font-awesome-picker>
                    </div>

                    <div v-if="form.id !== null"
                         v-can="'is_admin'"
                         class="form-group">
                        <MediumForm :form="form"
                                    :id="component_id"
                                    :medium_id="form.medium_id"
                                    :referenceable_type="'App\\PlanEntry'"
                                    :referenceable_id="form.id"
                                    accept="image/*"/>
                    </div>
                    <button :name="'planEntrySave'"
                            class="btn btn-primary p-2 m-2"
                            @click="submit()"
                    >
                        {{ trans('global.save') }}
                    </button>
                </div>

            </div>
        </div>
    </div>
</template>

<script>
/*const Calendar =
    () => import('../calendar/Calendar');*/

import Form from "form-backend-validation";
import MediumForm from "../media/MediumForm";
import Objectives from "../objectives/Objectives";
import FontAwesomePicker from "../../../views/forms/input/FontAwesomePicker";

const Trainings =
    () => import('../training/Trainings');

export default {
    props: {
        entry: {
            default: null
        },
        create: {
            default: false
        },
        plan: [],
        editable: {
            default: false
        },
        showTools: {
            default: false
        },
    },
    data() {
        return {
            component_id: this._uid,
            method: 'post',
            requestUrl: '/planEntries',
            form: new Form({
                'id': null,
                'title':'',
                'description': '',
                'plan_id': '',
                'css_icon': 'fas fa-calendar-day',
                'order_id': 0,
                'color': '#27AF60',
                'medium_id': null,

            }),
            editor: false,
            errors: {},
            owner_id: ''
        }
    },
    mounted() {
        this.owner_id = this.plan.owner_id;

        if ( this.entry !== null ) {
            this.form.id = this.entry.id;
            this.form.title = this.entry.title;
            this.form.description = this.entry.description;
            this.form.medium_id = this.entry.medium_id;
            this.form.css_icon = this.entry.css_icon;
            this.form.order_id = this.entry.order_id;

            this.method = 'patch';
        }
        this.form.plan_id = this.plan.id;

        // Set eventlistener for Media
        this.$eventHub.$on('addMedia', (e) => {
            if (this.component_id == e.id) {
                this.form.medium_id = e.selectedMediumId;
                if ( Array.isArray(this.form.medium_id))  {
                    this.form.medium_id = this.form.medium_id[0]; //Hack to get existing files working.
                }
            }
        });
    },
    methods: {
        setIcon(selectedIcon) {
            this.form.css_icon = 'fa fa-'+  selectedIcon.className;
        },
        edit() {
            this.editor = !this.editor;
            if (this.entry !== null) {
                this.form.color = this.entry.color;
                this.form.css_icon = this.entry.css_icon;
                this.form.order_id = this.plan.entries?.length ;
            }

            this.$nextTick(() => {
                this.$initTinyMCE("autolink link");
            });
        },
        submit() {
            let method = this.method.toLowerCase();
            this.form.description = tinyMCE.get('description').getContent();
            if (method === 'patch') {
                axios.patch(this.requestUrl + '/' + this.form.id, this.form)
                    .then(res => { // Tell the parent component we've updated a task
                        this.$eventHub.$emit("plan_entry_updated", res.data.entry);
                    })
                    .catch(error => { // Handle the error returned from our request
                        console.log(error);
                    });
            } else {
                axios.post(this.requestUrl, this.form)
                    .then(res => { // Tell the parent component we've added a new task and include it
                        this.$eventHub.$emit("plan_entry_added", res.data.entry);
                        this.form.reset(); // clear input-fields
                    })
                    .catch(error => { // Handle the error returned from our request
                        console.log(error);
                    });
            }
            this.editor = false;

        },
    },

    components: {
        FontAwesomePicker,
        Objectives,
        MediumForm,
        Trainings,
    },
}
</script>
<style scoped>
.card-header:hover {
    background-color: #e9ecef;
    cursor: pointer;
}
.card-header .fa-angle-up {
    transition: 0.3s transform;
}
.card-header.collapsed .fa-angle-up {
    transform: rotate(-180deg);
}
</style>
