<template>
	<div class="paramter-input-list-wrapper">
		<div v-for="parameter in filteredParameters" :key="parameter.name" :class="{indent}">
			<div
				v-if="multipleValues(parameter) === true && parameter.type !== 'fixedCollection'"
				class="parameter-item"
			>
				<multiple-parameter
					:parameter="parameter"
					:values="getParameterValue(nodeValues, parameter.name, path)"
					:nodeValues="nodeValues"
					:path="getPath(parameter.name)"
					@valueChanged="valueChanged"
				/>
			</div>

			<div v-else-if="parameter.type === 'notice'" class="parameter-item parameter-notice">
				<n8n-text size="small">
					<span v-html="$locale.nodeText().topParameterDisplayName(parameter)"></span>
				</n8n-text>
			</div>

			<div
				v-else-if="['collection', 'fixedCollection'].includes(parameter.type)"
				class="multi-parameter"
			>
				<div class="delete-option clickable" :title="$locale.baseText('parameterInputList.delete')" v-if="hideDelete !== true && !isReadOnly">
					<font-awesome-icon
						icon="trash"
						class="reset-icon clickable"
						:title="$locale.baseText('parameterInputList.parameterOptions')"
						@click="deleteOption(parameter.name)"
					/>
				</div>
				<n8n-input-label
					:label="$locale.nodeText().topParameterDisplayName(parameter)"
					:tooltipText="$locale.nodeText().topParameterDescription(parameter)"
					size="small"
					:underline="true"
					:labelHoverableOnly="true"
				>
					<collection-parameter
						v-if="parameter.type === 'collection'"
						:parameter="parameter"
						:values="getParameterValue(nodeValues, parameter.name, path)"
						:nodeValues="nodeValues"
						:path="getPath(parameter.name)"
						@valueChanged="valueChanged"
					/>
					<fixed-collection-parameter
						v-else-if="parameter.type === 'fixedCollection'"
						:parameter="parameter"
						:values="getParameterValue(nodeValues, parameter.name, path)"
						:nodeValues="nodeValues"
						:path="getPath(parameter.name)"
						@valueChanged="valueChanged"
					/>
				</n8n-input-label>
			</div>

			<div v-else-if="displayNodeParameter(parameter)" class="parameter-item">
				<div class="delete-option clickable" :title="$locale.baseText('parameterInputList.delete')" v-if="hideDelete !== true && !isReadOnly">
					<font-awesome-icon
						icon="trash"
						class="reset-icon clickable"
						:title="$locale.baseText('parameterInputList.deleteParameter')"
						@click="deleteOption(parameter.name)"
					/>
				</div>

				<parameter-input-full
					:parameter="parameter"
					:value="getParameterValue(nodeValues, parameter.name, path)"
					:displayOptions="true"
					:path="getPath(parameter.name)"
					:isReadOnly="isReadOnly"
					@valueChanged="valueChanged"
				/>
			</div>
		</div>
	</div>
</template>

<script lang="ts">

import {
	INodeParameters,
	INodeProperties,
	NodeParameterValue,
} from 'n8n-workflow';

import { IUpdateInformation } from '@/Interface';

import MultipleParameter from '@/components/MultipleParameter.vue';
import { genericHelpers } from '@/components/mixins/genericHelpers';
import { workflowHelpers } from '@/components/mixins/workflowHelpers';
import ParameterInputFull from '@/components/ParameterInputFull.vue';

import { get, set } from 'lodash';

import mixins from 'vue-typed-mixins';

export default mixins(
	genericHelpers,
	workflowHelpers,
)
	.extend({
		name: 'ParameterInputList',
		components: {
			MultipleParameter,
			ParameterInputFull,
		},
		props: [
			'nodeValues', // INodeParameters
			'parameters', // INodeProperties
			'path', // string
			'hideDelete', // boolean
			'indent',
		],
		computed: {
			filteredParameters (): INodeProperties[] {
				return this.parameters.filter((parameter: INodeProperties) => this.displayNodeParameter(parameter));
			},
			filteredParameterNames (): string[] {
				return this.filteredParameters.map(parameter => parameter.name);
			},
		},
		methods: {
			multipleValues (parameter: INodeProperties): boolean {
				if (this.getArgument('multipleValues', parameter) === true) {
					return true;
				}
				return false;
			},
			getArgument (
				argumentName: string,
				parameter: INodeProperties,
			): string | string[] | number | boolean | undefined{
				if (parameter.typeOptions === undefined) {
					return undefined;
				}

				if (parameter.typeOptions[argumentName] === undefined) {
					return undefined;
				}

				return parameter.typeOptions[argumentName];
			},
			getPath (parameterName: string): string {
				return (this.path ? `${this.path}.` : '') + parameterName;
			},
			deleteOption (optionName: string): void {
				const parameterData = {
					name: this.getPath(optionName),
					value: undefined,
				};

				// TODO: If there is only one option it should delete the whole one

				this.$emit('valueChanged', parameterData);
			},
			displayNodeParameter (parameter: INodeProperties): boolean {
				if (parameter.type === 'hidden') {
					return false;
				}

				if (parameter.displayOptions === undefined) {
					// If it is not defined no need to do a proper check
					return true;
				}

				const nodeValues: INodeParameters = {};
				let rawValues = this.nodeValues;
				if (this.path) {
					rawValues = get(this.nodeValues, this.path);
				}

				// Resolve expressions
				const resolveKeys = Object.keys(rawValues);
				let key: string;
				let i = 0;
				let parameterGotResolved = false;
				do {
					key = resolveKeys.shift() as string;
					if (typeof rawValues[key] === 'string' && rawValues[key].charAt(0) === '=') {
						// Contains an expression that
						if (rawValues[key].includes('$parameter') && resolveKeys.some(parameterName => rawValues[key].includes(parameterName))) {
							// Contains probably an expression of a missing parameter so skip
							resolveKeys.push(key);
							continue;
						} else {
							// Contains probably no expression with a missing parameter so resolve
							try {
								nodeValues[key] = this.resolveExpression(rawValues[key], nodeValues) as NodeParameterValue;
							} catch (e) {
								// If expression is invalid ignore
								nodeValues[key] = '';
							}
							parameterGotResolved = true;
						}
					} else {
						// Does not contain an expression, add directly
						nodeValues[key] = rawValues[key];
					}
					// TODO: Think about how to calculate this best
					if (i++ > 50) {
						// Make sure we do not get caught
						break;
					}
				} while(resolveKeys.length !== 0);

				if (parameterGotResolved === true) {
					if (this.path) {
						rawValues = JSON.parse(JSON.stringify(this.nodeValues));
						set(rawValues, this.path, nodeValues);
						return this.displayParameter(rawValues, parameter, this.path);
					} else {
						return this.displayParameter(nodeValues, parameter, '');
					}
				}

				return this.displayParameter(this.nodeValues, parameter, this.path);
			},
			valueChanged (parameterData: IUpdateInformation): void {
				this.$emit('valueChanged', parameterData);
			},
		},
		watch: {
			filteredParameterNames(newValue, oldValue) {
				if (newValue === undefined) {
					return;
				}
				// After a parameter does not get displayed anymore make sure that its value gets removed
				// Is only needed for the edge-case when a parameter gets displayed depending on another field
				// which contains an expression.
				for (const parameter of oldValue) {
					if (!newValue.includes(parameter)) {
						const parameterData = {
							name: `${this.path}.${parameter}`,
							node: this.$store.getters.activeNode.name,
							value: undefined,
						};
						this.$emit('valueChanged', parameterData);
					}
				}
			},
		},
		beforeCreate: function () { // tslint:disable-line
		// Because we have a circular dependency on CollectionParameter import it here
		// to not break Vue.
		this.$options!.components!.FixedCollectionParameter = require('./FixedCollectionParameter.vue').default;
		this.$options!.components!.CollectionParameter = require('./CollectionParameter.vue').default;
		},
	});
</script>

<style lang="scss">
.paramter-input-list-wrapper {
	.delete-option {
		display: none;
		position: absolute;
		z-index: 999;
		color: #f56c6c;
		font-size: var(--font-size-2xs);

		&:hover {
			color: #ff0000;
		}
	}

	.indent > div {
		padding-left: var(--spacing-s);
	}

	.multi-parameter {
		position: relative;
		margin: var(--spacing-xs) 0;

		.delete-option {
			top: 0;
			left: 0;
		}

		.parameter-info {
			display: none;
		}
	}

	.parameter-item {
		position: relative;
		margin: var(--spacing-xs) 0;

		>.delete-option {
			top: var(--spacing-5xs);
			left: 0;
		}
	}
	.parameter-item:hover > .delete-option,
	.multi-parameter:hover > .delete-option {
		display: block;
	}

	.parameter-notice {
		background-color: #fff5d3;
		color: $--custom-font-black;
		margin: 0.3em 0;
		padding: 0.7em;

		a {
			font-weight: var(--font-weight-bold);
		}
	}
}

</style>
