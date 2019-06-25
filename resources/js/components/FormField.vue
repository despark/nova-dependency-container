<template>
    <div v-if="dependenciesSatisfied">
        <div v-for="childField in field.fields">
            <component
                    :is="'form-' + childField.component"
                    :errors="errors"
                    :resource-id="resourceId"
                    :resource-name="resourceName"
                    :field="childField"
                    :ref="'field-' + childField.attribute"
            />
        </div>
    </div>
</template>

<script>
    import {FormField, HandlesValidationErrors} from 'laravel-nova'

    export default {
        mixins: [FormField, HandlesValidationErrors],

        props: ['resourceName', 'resourceId', 'field'],

        mounted() {
            this.registerDependencyWatchers(this.$root)
            this.updateDependencyStatus()
        },

        data() {
            return {
                dependencyValues: {},
                dependenciesSatisfied: false,
            }
        },

        methods: {

            registerDependencyWatchers(root) {
                root.$children.forEach(component => {
                    if (this.componentIsDependency(component)) {

                        component.$watch('value', (value) => {
                            this.dependencyValues[component.field.attribute] = value;
                            this.updateDependencyStatus()
                        }, {immediate: true})

                        this.dependencyValues[component.field.attribute] = component.field.value;

                    }

                    this.registerDependencyWatchers(component)
                })
            },

            componentIsDependency(component) {
                if (component.field === undefined) {
                    return false;
                }

                for (let dependency of this.field.dependencies) {
                    if (component.field.attribute === dependency.field) {
                        return true;
                    }
                }

                return false;
            },

            updateDependencyStatus() {
                for (let dependency of this.field.dependencies) {
                    if (dependency.hasOwnProperty('notEmpty') && !this.dependencyValues[dependency.field]) {
                        this.dependenciesSatisfied = false;
                        return;
                    }

                    if (dependency.hasOwnProperty('value') && typeof this.dependencyValues[dependency.field] === 'object' && typeof dependency.value === 'object' && this.dependencyValues[dependency.field] && this.dependencyValues[dependency.field].length) {
                        for (let i = 0; i < dependency.value.length; i++) {
                            let firstObject = dependency.value[i];

                            for (let j = 0; j < this.dependencyValues[dependency.field].length; j++) {
                                let secondObject = this.dependencyValues[dependency.field][j];

                                for (let property in firstObject) {
                                    if (firstObject.hasOwnProperty(property) && secondObject.hasOwnProperty(property) && firstObject[property] === secondObject[property]) {
                                        this.dependenciesSatisfied = true;
                                        return;
                                    } else {
                                        this.dependenciesSatisfied = false;
                                    }
                                }
                            }
                        }

                        return;
                    }

                    if (dependency.hasOwnProperty('value') && this.dependencyValues[dependency.field] !== dependency.value) {
                        this.dependenciesSatisfied = false;
                        return;
                    }
                }

                this.dependenciesSatisfied = true;
            },

            fill(formData) {
                if (this.dependenciesSatisfied) {
                    _.each(this.field.fields, field => {
                        if (field.fill) {
                            field.fill(formData)
                        }
                    })
                }
            }
        }
    }
</script>
