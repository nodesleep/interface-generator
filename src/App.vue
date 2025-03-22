<script setup lang="ts">
import { ref, reactive, computed } from "vue";

interface ConversionOptions {
  detectEnums: boolean;
  camelCase: boolean;
  markOptional: boolean;
  strictNullChecks: boolean;
}

type JsonValue =
  | string
  | number
  | boolean
  | null
  | JsonValue[]
  | { [key: string]: JsonValue };

interface InterfaceProperties {
  [key: string]: string;
}

const jsonInput = ref<string>("");
const jsonError = ref<string>("");
const typescriptOutput = ref<string>("");
const interfaceName = ref<string>("RootObject");
const copied = ref<boolean>(false);

const options = reactive<ConversionOptions>({
  detectEnums: true,
  camelCase: false,
  markOptional: false,
  strictNullChecks: true,
});

const isValidJson = computed<boolean>(() => {
  return jsonInput.value !== "" && !jsonError.value;
});

function validateJsonInput(): void {
  if (!jsonInput.value) {
    jsonError.value = "";
    return;
  }

  try {
    JSON.parse(jsonInput.value);
    jsonError.value = "";
  } catch (error) {
    if (error instanceof Error) {
      jsonError.value = `Invalid JSON: ${error.message}`;
    } else {
      jsonError.value = "Invalid JSON: Unknown error";
    }
  }
}

function generateTypeScript(): void {
  try {
    const parsed = JSON.parse(jsonInput.value) as JsonValue;
    typescriptOutput.value = convertJsonToTypeScript(
      parsed,
      interfaceName.value
    );
  } catch (error) {
    if (error instanceof Error) {
      jsonError.value = `Error generating TypeScript: ${error.message}`;
    } else {
      jsonError.value = "Error generating TypeScript: Unknown error";
    }
  }
}

function copyToClipboard(): void {
  navigator.clipboard.writeText(typescriptOutput.value);
  copied.value = true;
  setTimeout(() => {
    copied.value = false;
  }, 2000);
}

function loadSampleJson(): void {
  jsonInput.value = JSON.stringify(
    {
      id: 1,
      name: "John Doe",
      email: "john@example.com",
      is_active: true,
      score: 85.5,
      roles: ["admin", "user", "editor"],
      address: {
        street: "123 Main St",
        city: "Boston",
        zip: "02108",
        coordinates: {
          lat: 42.3601,
          lng: -71.0589,
        },
      },
      tags: ["important", "customer"],
      metadata: null,
      registration_date: "2023-01-15T08:30:00Z",
    },
    null,
    2
  );

  validateJsonInput();
}

// Core conversion logic
function convertJsonToTypeScript(json: JsonValue, rootName: string): string {
  const interfaces = new Map<string, InterfaceProperties>();
  const enumTypes = new Map<string, string[]>();

  // Generate the root interface - calling without storing the unused return value
  generateTypeDefinition(json, rootName, interfaces, enumTypes);

  // Combine all interfaces and enums into a single output
  let output = "";

  // Add enums first
  if (options.detectEnums) {
    for (const [enumName, enumValues] of enumTypes) {
      output += `enum ${enumName} {\n`;
      enumValues.forEach((value, index) => {
        const enumKey = value
          .replace(/[^a-zA-Z0-9_]/g, "_")
          .replace(/^[0-9]/, "_$&");
        output += `  ${enumKey} = "${value}"${
          index < enumValues.length - 1 ? "," : ""
        }\n`;
      });
      output += "}\n\n";
    }
  }

  // Add interfaces
  for (const [name, properties] of interfaces) {
    output += `interface ${name} {\n`;
    for (const [key, type] of Object.entries(properties)) {
      const formattedKey = formatPropertyKey(key);
      const optional = options.markOptional ? "?" : "";
      output += `  ${formattedKey}${optional}: ${type};\n`;
    }
    output += "}\n\n";
  }

  return output.trim();
}

function generateTypeDefinition(
  value: JsonValue,
  name: string,
  interfaces: Map<string, InterfaceProperties>,
  enumTypes: Map<string, string[]>,
  path = ""
): string {
  if (value === null) {
    return options.strictNullChecks ? "null" : "any";
  }

  // Handle different types
  if (Array.isArray(value)) {
    if (value.length === 0) {
      return "any[]";
    }

    // Check for potential enum (array of strings)
    if (
      options.detectEnums &&
      value.length > 0 &&
      value.every((item) => typeof item === "string")
    ) {
      const enumName = `${name}Enum`;
      const stringItems = value.filter(
        (item) => typeof item === "string"
      ) as string[];
      enumTypes.set(enumName, [...new Set(stringItems)]);
      return enumName;
    }

    // Find common type for array items
    const itemTypes = value.map((item) =>
      generateTypeDefinition(
        item,
        `${name}Item`,
        interfaces,
        enumTypes,
        `${path}.items`
      )
    );

    // If all items have the same type, use that type
    if (new Set(itemTypes).size === 1) {
      return `${itemTypes[0]}[]`;
    }

    // Otherwise, use union type
    return `(${[...new Set(itemTypes)].join(" | ")})[]`;
  }

  // Handle object type
  if (typeof value === "object") {
    const interfaceName = name.replace(/[^a-zA-Z0-9_]/g, "");
    const properties: InterfaceProperties = {};

    for (const [key, val] of Object.entries(value)) {
      const propName = options.camelCase ? toCamelCase(key) : key;
      const propType = generateTypeDefinition(
        val,
        `${interfaceName}${capitalizeFirstLetter(propName)}`,
        interfaces,
        enumTypes,
        `${path}.${propName}`
      );

      properties[key] = propType;
    }

    interfaces.set(interfaceName, properties);
    return interfaceName;
  }

  // Handle primitive types
  switch (typeof value) {
    case "string":
      return "string";
    case "number":
      // Simplified return statement - no need for ternary that returns the same value
      return "number";
    case "boolean":
      return "boolean";
    default:
      return "any";
  }
}

function formatPropertyKey(key: string): string {
  let formattedKey = options.camelCase ? toCamelCase(key) : key;

  // If the key has special characters, wrap it in quotes
  if (!/^[a-zA-Z_$][a-zA-Z0-9_$]*$/.test(formattedKey)) {
    formattedKey = `'${formattedKey}'`;
  }

  return formattedKey;
}

function toCamelCase(str: string): string {
  return str.replace(/_([a-z])/g, (_, letter) => letter.toUpperCase());
}

function capitalizeFirstLetter(string: string): string {
  return string.charAt(0).toUpperCase() + string.slice(1);
}
</script>

<template>
  <div
    class="min-h-screen bg-gray-50 text-gray-900 dark:bg-gray-900 dark:text-gray-100"
  >
    <div class="bg-sky-600 py-2 text-xs text-center">
      <a
        href="https://buymeacoffee.com/nodesleep"
        class="text-white font-medium hover:underline transition-all"
      >
        If you found this app useful, consider buying me a coffee by clicking
        here.
      </a>
    </div>
    <div class="max-w-6xl mx-auto py-8 px-4">
      <h1
        class="text-3xl font-bold mb-8 text-center text-black dark:text-sky-400"
      >
        JSON to TypeScript Interface Generator
      </h1>

      <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
        <!-- Input Section -->
        <div
          class="bg-white dark:bg-gray-800 rounded-lg shadow-md p-6 border border-gray-200 dark:border-gray-700"
        >
          <h2 class="text-xl font-semibold mb-4 text-sky-600 dark:text-sky-400">
            JSON Input
          </h2>

          <textarea
            v-model="jsonInput"
            placeholder="Paste your JSON here..."
            class="w-full h-64 p-3 bg-gray-50 dark:bg-gray-800 border border-gray-300 dark:border-gray-700 rounded-md font-mono text-sm text-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-sky-500 focus:border-transparent transition-all duration-300 resize-none"
            @input="validateJsonInput"
          ></textarea>

          <p v-if="jsonError" class="mt-2 text-red-500 text-sm">
            {{ jsonError }}
          </p>

          <div class="mt-6 space-y-4">
            <h3 class="font-medium text-sky-600 dark:text-sky-400">Options</h3>

            <div class="flex flex-col space-y-3">
              <label class="flex items-center space-x-2 cursor-pointer">
                <input
                  type="checkbox"
                  v-model="options.detectEnums"
                  class="rounded bg-gray-100 dark:bg-gray-700 border-gray-300 dark:border-gray-600 text-sky-500 focus:ring-sky-500 focus:ring-offset-2 dark:focus:ring-offset-gray-800"
                />
                <span>Detect and generate enums (for string arrays)</span>
              </label>

              <label class="flex items-center space-x-2 cursor-pointer">
                <input
                  type="checkbox"
                  v-model="options.camelCase"
                  class="rounded bg-gray-100 dark:bg-gray-700 border-gray-300 dark:border-gray-600 text-sky-500 focus:ring-sky-500 focus:ring-offset-2 dark:focus:ring-offset-gray-800"
                />
                <span>Convert keys to camelCase</span>
              </label>

              <label class="flex items-center space-x-2 cursor-pointer">
                <input
                  type="checkbox"
                  v-model="options.markOptional"
                  class="rounded bg-gray-100 dark:bg-gray-700 border-gray-300 dark:border-gray-600 text-sky-500 focus:ring-sky-500 focus:ring-offset-2 dark:focus:ring-offset-gray-800"
                />
                <span>Mark properties as optional</span>
              </label>

              <label class="flex items-center space-x-2 cursor-pointer">
                <input
                  type="checkbox"
                  v-model="options.strictNullChecks"
                  class="rounded bg-gray-100 dark:bg-gray-700 border-gray-300 dark:border-gray-600 text-sky-500 focus:ring-sky-500 focus:ring-offset-2 dark:focus:ring-offset-gray-800"
                />
                <span>Use strict null checks</span>
              </label>

              <label class="flex items-center space-x-2">
                <span>Root interface name:</span>
                <input
                  type="text"
                  v-model="interfaceName"
                  class="bg-gray-50 dark:bg-gray-700 border border-gray-300 dark:border-gray-600 rounded px-3 py-1 text-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-sky-500 focus:border-transparent transition-all duration-300"
                />
              </label>
            </div>
          </div>

          <button
            @click="generateTypeScript"
            class="mt-6 px-4 py-2 bg-sky-600 hover:bg-sky-700 text-white rounded-md focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-sky-500 disabled:opacity-50 transition-all duration-300 shadow-sm font-medium"
            :disabled="!isValidJson"
          >
            Generate TypeScript
          </button>
        </div>

        <!-- Output Section -->
        <div
          class="bg-white dark:bg-gray-800 rounded-lg shadow-md p-6 border border-gray-200 dark:border-gray-700"
        >
          <div class="flex justify-between items-center mb-4">
            <h2 class="text-xl font-semibold text-sky-600 dark:text-sky-400">
              TypeScript Output
            </h2>

            <button
              @click="copyToClipboard"
              class="px-3 py-1 text-sm bg-gray-100 dark:bg-gray-700 text-gray-800 dark:text-gray-200 rounded-md hover:bg-gray-200 dark:hover:bg-gray-600 focus:outline-none focus:ring-2 focus:ring-sky-500 transition-colors duration-300 flex items-center space-x-1"
              v-if="typescriptOutput"
            >
              <span>{{ copied ? "Copied!" : "Copy" }}</span>
            </button>
          </div>

          <pre
            v-if="typescriptOutput"
            class="bg-gray-50 dark:bg-gray-800 p-4 rounded-md border border-gray-200 dark:border-gray-700 h-64 overflow-auto font-mono text-sm text-gray-800 dark:text-gray-200 scrollbar"
            >{{ typescriptOutput }}</pre
          >

          <div
            v-else
            class="bg-gray-50 dark:bg-gray-800 p-4 rounded-md border border-gray-200 dark:border-gray-700 h-64 flex items-center justify-center text-gray-500 dark:text-gray-400"
          >
            Generate TypeScript to see the output here
          </div>
        </div>
      </div>

      <!-- Sample JSON Button -->
      <div class="mt-8 text-center">
        <button
          @click="loadSampleJson"
          class="px-4 py-2 bg-gray-200 dark:bg-gray-700 text-gray-800 dark:text-gray-200 rounded-md hover:bg-gray-300 dark:hover:bg-gray-600 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500 transition-all duration-300 shadow-sm"
        >
          Load Sample JSON
        </button>
      </div>
    </div>
  </div>
</template>

<style>
.scrollbar::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

.scrollbar::-webkit-scrollbar-track {
  background: #e5e7eb;
  border-radius: 4px;
}

.dark .scrollbar::-webkit-scrollbar-track {
  background: #374151;
}

.scrollbar::-webkit-scrollbar-thumb {
  background: #94a3b8;
  border-radius: 4px;
}

.dark .scrollbar::-webkit-scrollbar-thumb {
  background: #4b5563;
}

.scrollbar::-webkit-scrollbar-thumb:hover {
  background: #64748b;
}

.dark .scrollbar::-webkit-scrollbar-thumb:hover {
  background: #6b7280;
}
</style>
