<template>
    <v-dialog
        v-model="editPropertyDialog"
        width="860"
        persistent
        @keydown="keyDown"
        @keyup="keyUp($event, saveProperty)">
        <v-card>
            <v-card-title>
                <span class="text-subtile-1">{{ props.newProperty ? 'Create a new Property' : 'Edit Property' }}</span>
            </v-card-title>
            <v-divider></v-divider>
            <v-card-text style="overflow-y: auto" class="pa-3 bg-card">
                <v-expansion-panels v-model="openPanels" multiple>
                    <!-- Details -->
                    <v-expansion-panel class="border-t-thin border-s-thin border-e-thin" :class="bordersToShow(0)">
                        <v-expansion-panel-title>Details</v-expansion-panel-title>
                        <v-expansion-panel-text>
                            <TextInput
                                v-model="propertyIdShort"
                                label="IdShort"
                                :error="hasError('idShort')"
                                :rules="[rules.required]"
                                :error-messages="getError('idShort')" />
                            <MultiLanguageTextInput
                                v-model="displayName"
                                :show-label="true"
                                label="Display Name"
                                type="displayName" />
                            <MultiLanguageTextInput
                                v-model="description"
                                :show-label="true"
                                label="Description"
                                type="description" />
                            <SelectInput
                                v-model="propertyCategory"
                                label="Category"
                                type="category"
                                :clearable="true" />
                        </v-expansion-panel-text>
                    </v-expansion-panel>
                    <!-- TODO: Value ID -->
                    <!-- Property Value -->
                    <v-expansion-panel class="border-s-thin border-e-thin" :class="bordersToShow(1)">
                        <v-expansion-panel-title>Value</v-expansion-panel-title>
                        <v-expansion-panel-text>
                            <SelectInput v-model="valueType" label="Data Type" type="dataType" :clearable="true" />
                            <TextInput v-model="propertyValue" label="Value" />
                        </v-expansion-panel-text>
                    </v-expansion-panel>
                    <!-- Semantic ID -->
                    <v-expansion-panel class="border-s-thin border-e-thin" :class="bordersToShow(2)">
                        <v-expansion-panel-title>Semantic ID</v-expansion-panel-title>
                        <v-expansion-panel-text>
                            <ReferenceInput v-model="semanticId" label="Semantic ID" :no-header="true" />
                        </v-expansion-panel-text>
                    </v-expansion-panel>
                    <!-- Data Specification -->
                    <v-expansion-panel class="border-b-thin border-s-thin border-e-thin" :class="bordersToShow(3)">
                        <v-expansion-panel-title>Data Specification</v-expansion-panel-title>
                        <v-expansion-panel-text>
                            <span class="text-subtitleText text-subtitle-2">Coming soon!</span>
                        </v-expansion-panel-text>
                    </v-expansion-panel>
                </v-expansion-panels>
            </v-card-text>
            <v-divider></v-divider>
            <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn @click="closeDialog">Cancel</v-btn>
                <v-btn color="primary" @click="saveProperty">Save</v-btn>
            </v-card-actions>
        </v-card>
    </v-dialog>
</template>

<script setup lang="ts">
    /*
        NOTE: This component uses Keyboard events (keyUp,keyDown) in the root element v-dialog.
        It saves the changes after pressing the 'Enter' Key. When creating additional Form Inputs that require or support the
        usage of the 'Enter' key, make sure to edit the keyDown/keyUp method to not execute when in such form fields.
    */

    import { jsonization, types as aasTypes } from '@aas-core-works/aas-core3.0-typescript';
    import { computed, ref, watch } from 'vue';
    import { useRoute, useRouter } from 'vue-router';
    import { useSMRepositoryClient } from '@/composables/Client/SMRepositoryClient';
    import { useAASStore } from '@/store/AASDataStore';
    import { useNavigationStore } from '@/store/NavigationStore';
    import { extractEndpointHref } from '@/utils/AAS/DescriptorUtils';
    import { keyDown, keyUp } from '@/utils/EditorUtils';
    import { base64Decode } from '@/utils/EncodeDecodeUtils';

    const props = defineProps<{
        modelValue: boolean;
        newProperty: boolean;
        parentElement: any;
        path?: string;
        property?: any;
    }>();

    // Stores
    const navigationStore = useNavigationStore();
    const aasStore = useAASStore();

    // Composables
    const { fetchSme, putSubmodelElement, postSubmodelElement } = useSMRepositoryClient();

    // Vue Router
    const router = useRouter();
    const route = useRoute();

    const editPropertyDialog = ref(false);
    const propertyObject = ref<aasTypes.Property | undefined>(undefined);
    const openPanels = ref<number[]>([0]);

    const propertyIdShort = ref<string | null>(null);

    const displayName = ref<Array<aasTypes.LangStringNameType> | null>(null);
    const description = ref<Array<aasTypes.LangStringTextType> | null>(null);
    const propertyCategory = ref<string | null>(null);

    const semanticId = ref<aasTypes.Reference | null>(null);
    const propertyValue = ref<string | null>(null);
    const valueType = ref<aasTypes.DataTypeDefXsd>(aasTypes.DataTypeDefXsd.String);

    const errors = ref<Map<string, string>>(new Map());

    const rules = {
        required: (value: any) => !!value || 'Required.',
    };

    const emit = defineEmits<{
        (event: 'update:modelValue', value: boolean): void;
    }>();

    watch(
        () => props.modelValue,
        (value) => {
            editPropertyDialog.value = value;
            if (value) {
                initializeInputs();
            }
        }
    );

    watch(
        () => editPropertyDialog.value,
        (value) => {
            emit('update:modelValue', value);
        }
    );

    const selectedAAS = computed(() => aasStore.getSelectedAAS); // Get the selected AAS from Store

    const bordersToShow = computed(() => (panel: number) => {
        let border = '';
        switch (panel) {
            case 0:
                if (openPanels.value.includes(0) || openPanels.value.includes(1)) {
                    border = 'border-b-thin';
                }
                break;
            case 1:
                if (openPanels.value.includes(0) || openPanels.value.includes(1)) {
                    border += ' border-t-thin';
                }
                if (openPanels.value.includes(1) || openPanels.value.includes(2)) {
                    border += ' border-b-thin';
                }
                break;
            case 2:
                if (openPanels.value.includes(1) || openPanels.value.includes(2)) {
                    border += ' border-t-thin';
                }
                if (openPanels.value.includes(2) || openPanels.value.includes(3)) {
                    border += ' border-b-thin';
                }
                break;
            case 3:
                if (openPanels.value.includes(2) || openPanels.value.includes(3)) {
                    border += 'border-t-thin';
                }
                break;
        }
        return border;
    });

    function hasError(field: string): boolean {
        return errors.value.has(field);
    }

    function getError(field: string): string | undefined {
        if (!hasError(field)) {
            return undefined;
        }
        return errors.value.get(field);
    }

    async function saveProperty(): Promise<void> {
        if (props.newProperty || propertyObject.value === undefined) {
            propertyObject.value = new aasTypes.Property(valueType.value);
        }

        if (propertyIdShort.value !== null) {
            propertyObject.value.idShort = propertyIdShort.value;
        } else {
            errors.value.set('idShort', 'Property IdShort is required');
            return;
        }

        propertyObject.value.value = propertyValue.value;

        propertyObject.value.valueType = valueType.value;

        if (semanticId.value !== null) {
            propertyObject.value.semanticId = semanticId.value;
        }

        if (displayName.value !== null) {
            propertyObject.value.displayName = displayName.value;
        }

        if (description.value !== null) {
            propertyObject.value.description = description.value;
        }

        propertyObject.value.category = propertyCategory.value;

        if (props.newProperty) {
            if (props.parentElement.modelType === 'Submodel') {
                // Create the property on the parent Submodel
                await postSubmodelElement(propertyObject.value, props.parentElement.id);

                const aasEndpoint = extractEndpointHref(selectedAAS.value, 'AAS-3.0');

                // Navigate to the new property
                router.push({
                    query: {
                        aas: aasEndpoint,
                        path: props.parentElement.path + '/submodel-elements/' + propertyObject.value.idShort,
                    },
                });
            } else {
                // Extract the submodel ID and the idShortPath from the parentElement path
                const splitted = props.parentElement.path.split('/submodel-elements/');
                const submodelId = base64Decode(splitted[0].split('/submodels/')[1]);
                const idShortPath = splitted[1];

                // Create the property on the parent element
                await postSubmodelElement(propertyObject.value, submodelId, idShortPath);

                const aasEndpoint = extractEndpointHref(selectedAAS.value, 'AAS-3.0');

                // Navigate to the new property
                if (props.parentElement.modelType === 'SubmodelElementCollection') {
                    router.push({
                        query: {
                            aas: aasEndpoint,
                            path: props.parentElement.path + '.' + propertyObject.value.idShort,
                        },
                    });
                }
            }
        } else {
            if (props.path == undefined) {
                console.error('Property Path is missing');
                return;
            }

            const editedElementSelected = route.query.path === props.path;
            const aasEndpoint = extractEndpointHref(selectedAAS.value, 'AAS-3.0');

            // Update the property
            if (props.parentElement.modelType === 'Submodel') {
                await putSubmodelElement(propertyObject.value, props.path);

                if (editedElementSelected) {
                    router.push({
                        query: {
                            aas: aasEndpoint,
                            path: props.parentElement.path + '/submodel-elements/' + propertyObject.value.idShort,
                        },
                    });
                }
            } else if (props.parentElement.modelType === 'SubmodelElementList') {
                const index = props.parentElement.value.indexOf(
                    props.parentElement.value.find((el: any) => el.id === props.property.id)
                );
                const path = props.parentElement.path + `%5B${index}%5D`;
                await putSubmodelElement(propertyObject.value, path);

                if (editedElementSelected) {
                    router.push({
                        query: { aas: aasEndpoint, path: path },
                    });
                }
            } else {
                // Submodel Element Collection or Entity
                await putSubmodelElement(propertyObject.value, props.property.path);

                if (editedElementSelected) {
                    router.push({
                        query: {
                            aas: aasEndpoint,
                            path: props.parentElement.path + '.' + propertyObject.value.idShort,
                        },
                    });
                }
            }
        }
        closeDialog();
        navigationStore.dispatchTriggerTreeviewReload();
    }

    function closeDialog(): void {
        editPropertyDialog.value = false;
    }

    async function initializeInputs(): Promise<void> {
        if (!props.newProperty && props.property) {
            const propertyJSON = await fetchSme(props.property.path);
            const instanceOrError = jsonization.propertyFromJsonable(propertyJSON);

            if (instanceOrError.error !== null) {
                console.error('Error parsing Property: ', instanceOrError.error);
                return;
            }

            propertyObject.value = instanceOrError.mustValue();

            propertyIdShort.value = propertyObject.value.idShort;

            if (propertyObject.value.displayName) {
                displayName.value = propertyObject.value.displayName;
            }
            if (propertyObject.value.description) {
                description.value = propertyObject.value.description;
            }
            if (propertyObject.value.category) {
                propertyCategory.value = propertyObject.value.category;
            }
            if (propertyObject.value.value) {
                propertyValue.value = propertyObject.value.value;
            }
            if (propertyObject.value.valueType) {
                valueType.value = propertyObject.value.valueType;
            }
            if (propertyObject.value.semanticId) {
                semanticId.value = propertyObject.value.semanticId;
            }
            openPanels.value = [0, 1];
        } else {
            propertyIdShort.value = null;
            displayName.value = null;
            description.value = null;
            propertyCategory.value = null;
            propertyValue.value = null;
            valueType.value = aasTypes.DataTypeDefXsd.String;
            semanticId.value = null;
            openPanels.value = [0, 1];
        }
    }
</script>
