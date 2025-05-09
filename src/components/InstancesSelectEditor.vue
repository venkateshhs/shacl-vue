<template>
    <v-input
        v-model="internalValue"
        :rules="rules"
        ref="fieldRef"
        :id="inputId"
        hide-details="auto"
        style="margin-bottom: 1em;"
    >
        <v-autocomplete
            v-model="subValues.selectedInstance"
            variant="outlined"
            hide-details="auto"
            label="select an item"
            :items="itemsToList"
            validate-on="lazy input"
            item-value="value"
            item-title="title"
            return-object
            ref="editorComp"
            lines="two"
            :custom-filter="filterListItems"
        >

        <template v-slot:selection="data">
            
            <span v-if="data.item.props.hasPrefLabel">
                {{ data.item.props[toCURIE(SKOS.prefLabel.value, allPrefixes)] }}
            </span>
            <span v-else>
                <div style="transform: scale(0.6); margin: 0; padding: 0;">
                    <span v-for="(value, key, index) in data.item.props">
                        <span v-if="['title', 'subtitle', 'name', 'value', RDF.type.value, toCURIE(RDF.type.value, allPrefixes)].indexOf(key) < 0">
                            {{ key }}: {{ value }} <br>
                        </span>
                    </span>
                </div>
            </span>
            
        </template>

            <template v-slot:item="data">
                <!-- Show the "Add Item" button first -->
                <div v-if="data.item.props.isButton">
                    <v-list-item @click.stop :active="false">
                        <v-list-item-title>
                            <v-menu v-model="menu" location="end">
                                <template v-slot:activator="{ props }">
                                    <v-btn variant="tonal" v-bind="props">{{ data.item.title }} &nbsp;&nbsp; <v-icon icon="item.icon">mdi-play</v-icon></v-btn>
                                </template>

                                <v-list ref="addItemList">
                                    <v-list-item v-for="item in propClassList" @click.stop="handleAddItemClick(item)">
                                        <v-list-item-title>{{ item.title }}</v-list-item-title>
                                    </v-list-item>
                                </v-list>
                            </v-menu>
                        </v-list-item-title>
                    </v-list-item>
                </div>
                <!-- Then show all other items -->
                <div v-else>
                    <v-list-item @click.stop="selectItem(data.item)" rounded :active="subValues.selectedInstance?.value == data.item.value" class="myInstancesList">
                        <template v-slot:prepend>
                            <v-icon>{{ getClassIcon(toIRI(data.item.props[toCURIE(RDF.type.value, allPrefixes)], allPrefixes)) }}</v-icon>
                        </template>
                        <span v-if="data.item.props.hasPrefLabel">
                            {{ data.item.props[toCURIE(SKOS.prefLabel.value, allPrefixes)] }}
                        </span>
                        <span v-else>
                            <span v-for="(value, key, index) in data.item.props">
                                <v-row no-gutters v-if="['title', 'subtitle', 'name', 'value', RDF.type.value, toCURIE(RDF.type.value, allPrefixes)].indexOf(key) < 0">
                                    <v-col cols="6"><small>{{ key }}</small></v-col>
                                    <v-col><small>{{ value }}</small></v-col>
                                </v-row>
                            </span>
                        </span>
                    </v-list-item>
                    <v-divider></v-divider>
                    <v-divider></v-divider>
                </div>
            </template>
        </v-autocomplete>
    </v-input>
</template>

<script setup>
    import { inject, watch, onBeforeMount, onMounted, ref, provide, computed, nextTick} from 'vue'
    import { useRules } from '../composables/rules'
    import rdf from 'rdf-ext'
    import { SHACL, RDF, RDFS, SKOS } from '@/modules/namespaces'
    import { findObjectByKey } from '../modules/utils';
    import { toCURIE, toIRI } from 'shacl-tulip'
    import { useRegisterRef } from '../composables/refregister';
    import { useBaseInput } from '@/composables/base';

    // ----- //
    // Props //
    // ----- //

    const props = defineProps({
        modelValue: String,
        property_shape: Object,
        node_uid: String,
        node_idx: String,
        triple_uid: String,
        triple_idx: Number,
    })
    

    // ---- //
    // Data //
    // ---- //
    const itemsToList = ref([]);
    const rdfDS = inject('rdfDS');
    const allPrefixes = inject('allPrefixes');
    const classDS = inject('classDS');
    const config = inject('config');
    const fetchFromService = inject('fetchFromService');
    const localPropertyShape = ref(props.property_shape)
    const propClass = ref(null)
    propClass.value = localPropertyShape.value[SHACL.class.value] ?? false
    if (config.value.use_service) {
        const allclass_array = getAllClasses(propClass.value)
        await getAllRecordsFromService(allclass_array)
    }
    getItemsToList()
    const editorComp = ref(null)
    const { rules } = useRules(localPropertyShape.value)
    const inputId = `input-${Date.now()}`;
    const { fieldRef } = useRegisterRef(inputId, props);
    const emit = defineEmits(['update:modelValue']);
    const { subValues, internalValue } = useBaseInput(
        props,
        emit,
        valueParser,
        valueCombiner
    );

    const newNodeIdx = ref(null)
    const addItemList = ref(null)
    const menu = ref(false)
    const selectedAddItemShapeIRI = ref(null)
    const addForm = inject('addForm');
    const removeForm = inject('removeForm');
    const getClassIcon = inject('getClassIcon')
    // const activatedInstancesSelectEditor = inject('activatedInstancesSelectEditor')
    const lastSavedNode = inject('lastSavedNode')
    const openForms = inject('openForms')

    const cancelDialogForm = () => {
        // console.log("Canceling from form in dialog")
        newNodeIdx.value = null
    };
    provide('cancelFormHandler', cancelDialogForm);
    const saveDialogForm = () => {
        // console.log("Saving from form in dialog")
        newNodeIdx.value = null
    };
    provide('saveFormHandler', saveDialogForm);

    // ------------------- //
    // Computed properties //
    // ------------------- //

    // trigger whenever lastSavedNode is updated, i.e. whenever a form is saved 
    watch(lastSavedNode, (savedNode) => {
        if (savedNode) {
            if(!openForms.at(-1).activatedInstancesSelectEditor) {
                return;
            }
            // First check if the current component is also the activatedInstancesSelectEditor
            // If not, do nothing
            if (
                openForms.at(-1).activatedInstancesSelectEditor.nodeshape_iri == props.node_uid &&
                openForms.at(-1).activatedInstancesSelectEditor.node_iri == props.node_idx &&
                openForms.at(-1).activatedInstancesSelectEditor.predicate_iri == props.triple_uid &&
                openForms.at(-1).activatedInstancesSelectEditor.predicate_idx == props.triple_idx
            ) {
                // If this is the instancesSelectEditor instance that activated the recently saved
                // and closed form, we want to set the selected instance as the saved node
                // This is the format of savedNode:
                // {
                //     nodeshape_iri: nodeshape_iri,
                //     node_iri: subject_iri
                // }
                // First, let's make sure the list is updated to include the recently saved node:
                getItemsToList()
                // Then we need to set the selectedInstance. The itemsToList has items as objects,
                // with value = quad.subject.value. 
                // We need to find the item that has the value being the same as the saved node's node_iri:
                var inst = findObjectByKey(itemsToList.value, "value", savedNode.node_iri)
                if (inst) {
                    subValues.value.selectedInstance = inst
                } else {
                    console.log("A node was recently saved but it could not be found in the itemsToList and was therefore not set as the selectedItem")
                }
                openForms.at(-1).activatedInstancesSelectEditor = null
                lastSavedNode.value = null
            } else {
                console.log("activatedInstancesSelectEditor is not the same as last saved node")
            }
        }
    }, {immediate: true });


    watch(rdfDS.data.graph, () => {
        console.log("CHECK: graphdata instanceselecteditor")
        getItemsToList()
    }, { deep: true });

    const propClassList = computed(() => {
        var items = []
        // first add main property class
        items.push(
            {
                title: toCURIE(propClass.value, allPrefixes),
                value: propClass.value
            }
        )
        const subClasses = rdf.grapoi({ dataset: classDS.data.graph })
            .hasOut(rdf.namedNode(RDFS.subClassOf.value), rdf.namedNode(propClass.value))
            .quads();
        
        Array.from(subClasses).forEach(quad => {
            items.push(
                {
                    title: toCURIE(quad.subject.value, allPrefixes),
                    value: quad.subject.value
                }
            )
        });
        return items
    })

    

    // --------- //
    // Functions //
    // --------- //

    function valueParser(value) {
        // Parsing internalValue into ref values for separate subcomponent(s)
        if (!itemsToList.value) {
            return { selectedInstance: null };
        }
        var inst = findObjectByKey(itemsToList.value, "value", value)
        return { selectedInstance: inst ?? null }
    }

    function valueCombiner(values) {
        // Determine internalValue from subvalues/subcomponents
        return values.selectedInstance ? values.selectedInstance.value : null
    }

    function selectItem(item) {
        subValues.value.selectedInstance = item;
        editorComp.value.blur();
    }

    function handleAddItemClick(item) {
        openForms.at(-1).activatedInstancesSelectEditor = {
            nodeshape_iri: props.node_uid,
            node_iri: props.node_idx,
            predicate_iri: props.triple_uid,
            predicate_idx: props.triple_idx,
        }
        selectedAddItemShapeIRI.value = item.value
        newNodeIdx.value = '_:' + crypto.randomUUID()
        console.log("New form shape IRI")
        console.log(selectedAddItemShapeIRI.value)
        console.log("New form node IRI")
        console.log(newNodeIdx.value)
        menu.value = false;
        addForm(selectedAddItemShapeIRI.value, newNodeIdx.value, 'new')
    }

    function getAllClasses(main_class) {
        return [main_class].concat(getSubClasses(main_class))
    }


    function getSubClasses(main_class) {
        // Find quads in the subclass datasetnodes with predicate rdfs:subClassOf
        // object main_class, and return as an array of terms
        const subClasses = rdf.grapoi({ dataset: classDS.data.graph })
            .hasOut(rdf.namedNode(RDFS.subClassOf.value), rdf.namedNode(main_class))
            .quads();
        var myArr = []
        Array.from(subClasses).forEach(quad => {
            myArr.push(quad.subject.value)
        });
        return myArr
    }

    async function getAllRecordsFromService(iri_array) {
        for (const iri of iri_array) {
            const result = await fetchFromService('get-records', iri, allPrefixes)
        }
    }

    function filterListItems(itemTitle, queryText, item) {
        const searchText = queryText.toLowerCase()
        return item.raw.props.hasPrefLabel &&
            item.raw.props[toCURIE(SKOS.prefLabel.value, allPrefixes)].toLowerCase().indexOf(searchText) > -1
    }

    function getItemsToList() {
        // ---
        // The goal of this method is to populate the list of items for the
        // InstancesSelectEditor
        // ---
        // console.log("(Re)calculating instance items")
        // find nodes with predicate rdf:type and object being the property class
        // console.log("find nodes with predicate rdf:type and object being the property class:")
        var quads = rdfDS.getLiteralAndNamedNodes(rdf.namedNode(RDF.type), propClass.value, allPrefixes)
        // then find nodes with predicate rdfs:subClassOf and object being the property class
        // TODO: here we are only using a named node for the object because this is how the
        // tools/gen_owl_minimal.py script outputs the triples in the ttl file. This should be
        // generalised
        const subClasses = rdf.grapoi({ dataset: classDS.data.graph })
            .hasOut(rdf.namedNode(RDFS.subClassOf.value), rdf.namedNode(propClass.value))
            .quads();
        // For each subclass, find the quads in graphData that has the class name as object
        // and RDF.type as predicate
        var myArr = []
        Array.from(subClasses).forEach(quad => {
            const cl = quad.subject.value
            // console.log(`\t - getting quads with class: ${cl}`)
            // console.log(`\t - (size of data graph: ${graphData.size})`)
            myArr = myArr.concat(rdfDS.getLiteralAndNamedNodes(rdf.namedNode(RDF.type), cl, allPrefixes))
        });
        // Then combine all quad arrays
        // const combinedQuads = quads.concat(savedQuads).concat(myArr);
        const combinedQuads = quads.concat(myArr);
        // Finally, create list items from quads
        var itemsToListArr = []
        itemsToListArr.push(
            {
                title: "Add New Item",
                props: { isButton: true, },
            }
        )
        combinedQuads.forEach(quad => {
            var extra = ''
            if (quad.subject.termType === 'BlankNode') {
                extra = ' (BlankNode)'
            }
            var relatedTrips = rdfDS.getSubjectTriples(quad.subject)
            var item = {
                title: quad.subject.value + extra,
                value: quad.subject.value,
                props: { subtitle: toCURIE(quad.object.value, allPrefixes), hasPrefLabel: false },
            }
            relatedTrips.forEach(quad => {
                item.props[toCURIE(quad.predicate.value, allPrefixes)] = toCURIE(quad.object.value, allPrefixes)
                if (quad.predicate.value == SKOS.prefLabel.value) {
                    item.props.hasPrefLabel = true
                } 
            })
            itemsToListArr.push(item)
        });
        itemsToList.value = itemsToListArr
    }
</script>

<!-- Component matching logic -->

<script>
    import { SHACL } from '@/modules/namespaces'
    export const matchingLogic = (shape) => {
        // sh:nodeKind exists
        if ( shape.hasOwnProperty(SHACL.nodeKind.value) ) {
            // sh:nodeKind == sh:IRI ||
            // sh:nodeKind == sh:BlankNodeOrIRI ||
            return [SHACL.IRI.value, SHACL.BlankNodeOrIRI.value].includes(shape[SHACL.nodeKind.value])
        }
        return false
    };
</script>


<style scoped>
.info-tooltip {
  cursor: pointer;
}
</style>