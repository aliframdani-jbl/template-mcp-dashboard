<script setup>
import { ref, onMounted } from 'vue';
import { useToast } from 'primevue/usetoast';

// Simulated TokenService for CRUD operations
const TokenService = {
    getTokens() {
        // Simulate API call
        return Promise.resolve([
            { id: '1', name: 'Token 1', value: '3FjyRKABR0L8lnZPMMzD2URc9nRTPaZ4', description: 'First token', createdAt: new Date().toISOString() },
            { id: '2', name: 'Token 2', value: '3FjyRKABR0L8lnZPMMzD2URc9nRTPaZ4', description: 'Second token', createdAt: new Date().toISOString() }
        ]);
    }
    // Add more methods for real API integration
};

// Simulated PermissionService
const PermissionService = {
    getPermissions() {
        // Simulate API call
        return Promise.resolve({
            canCreate: true,
            canEdit: true,
            canDelete: true,
            canExport: true
        });
    }
};

const toast = useToast();
const dt = ref();
const tokens = ref([]);
const tokenDialog = ref(false);
const deleteTokenDialog = ref(false);
const deleteTokensDialog = ref(false);
const token = ref({});
const selectedTokens = ref([]);
const filters = ref({
    global: { value: null, matchMode: 'contains' }
});
const submitted = ref(false);

// For toggling token visibility
const tokenVisibility = ref({}); // { [id]: true/false }

// Permissions
const permissions = ref({
    canCreate: false,
    canEdit: false,
    canDelete: false,
    canExport: false
});

onMounted(async () => {
    tokens.value = await TokenService.getTokens();
    const perms = await PermissionService.getPermissions();
    permissions.value = perms;
    // Initialize all tokens as hidden
    tokens.value.forEach((t) => (tokenVisibility.value[t.id] = false));
});

function openNew() {
    token.value = {
        permissions: []
    };
    submitted.value = false;
    tokenDialog.value = true;
}

function hideDialog() {
    tokenDialog.value = false;
    submitted.value = false;
}

// Show permissions in toast when token is created
function saveToken() {
    submitted.value = true;
    if (token.value.name?.trim()) {
        if (token.value.id) {
            // Update
            const idx = findIndexById(token.value.id);
            tokens.value[idx] = { ...token.value };
            toast.add({ severity: 'success', summary: 'Successful', detail: 'Token Updated', life: 3000 });
        } else {
            // Create
            token.value.id = createId();
            token.value.value = generateToken();
            token.value.createdAt = new Date().toISOString();
            tokens.value.push({ ...token.value });
            // Set initial visibility to hidden
            tokenVisibility.value[token.value.id] = false;
            // Show permissions in toast
            const perms = (token.value.permissions || []).map((p) => p.charAt(0).toUpperCase() + p.slice(1)).join(', ') || 'None';
            toast.add({
                severity: 'success',
                summary: 'Token Created',
                detail: `Token Created with permissions: ${perms}`,
                life: 4000
            });
        }
        tokenDialog.value = false;
        token.value = {};
    }
}

function editToken(tok) {
    token.value = { ...tok };
    tokenDialog.value = true;
}

function confirmDeleteToken(tok) {
    token.value = tok;
    deleteTokenDialog.value = true;
}

function deleteToken() {
    tokens.value = tokens.value.filter((val) => val.id !== token.value.id);
    deleteTokenDialog.value = false;
    // Remove visibility state
    delete tokenVisibility.value[token.value.id];
    token.value = {};
    toast.add({ severity: 'success', summary: 'Successful', detail: 'Token Deleted', life: 3000 });
}

function findIndexById(id) {
    return tokens.value.findIndex((t) => t.id === id);
}

function createId() {
    let id = '';
    const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
    for (let i = 0; i < 8; i++) {
        id += chars.charAt(Math.floor(Math.random() * chars.length));
    }
    return id;
}

function generateToken() {
    let token = '';
    const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
    for (let i = 0; i < 32; i++) {
        token += chars.charAt(Math.floor(Math.random() * chars.length));
    }
    return token;
}

function exportCSV() {
    dt.value.exportCSV();
}

function confirmDeleteSelected() {
    deleteTokensDialog.value = true;
}

function deleteSelectedTokens() {
    selectedTokens.value.forEach((tok) => delete tokenVisibility.value[tok.id]);
    tokens.value = tokens.value.filter((val) => !selectedTokens.value.includes(val));
    deleteTokensDialog.value = false;
    selectedTokens.value = [];
    toast.add({ severity: 'success', summary: 'Successful', detail: 'Tokens Deleted', life: 3000 });
}

// Toggle token visibility
function toggleTokenVisibility(id) {
    tokenVisibility.value[id] = !tokenVisibility.value[id];
}
</script>
<template>
    <div>
        <div class="card">
            <Toolbar class="mb-6">
                <template #start>
                    <Button label="New" icon="pi pi-plus" severity="secondary" class="mr-2" @click="openNew" :disabled="!permissions.canCreate" />
                    <Button label="Delete" icon="pi pi-trash" severity="secondary" @click="confirmDeleteSelected" :disabled="!selectedTokens || !selectedTokens.length || !permissions.canDelete" />
                </template>
                <template #end>
                    <Button label="Export" icon="pi pi-upload" severity="secondary" @click="exportCSV($event)" :disabled="!permissions.canExport" />
                </template>
            </Toolbar>

            <DataTable
                ref="dt"
                v-model:selection="selectedTokens"
                :value="tokens"
                dataKey="id"
                :paginator="true"
                :rows="10"
                :filters="filters"
                paginatorTemplate="FirstPageLink PrevPageLink PageLinks NextPageLink LastPageLink CurrentPageReport RowsPerPageDropdown"
                :rowsPerPageOptions="[5, 10, 25]"
                currentPageReportTemplate="Showing {first} to {last} of {totalRecords} tokens"
            >
                <template #header>
                    <div class="flex flex-wrap gap-2 items-center justify-between">
                        <h4 class="m-0">Manage Tokens</h4>
                        <IconField>
                            <InputIcon>
                                <i class="pi pi-search" />
                            </InputIcon>
                            <InputText v-model="filters['global'].value" placeholder="Search..." />
                        </IconField>
                    </div>
                </template>

                <Column selectionMode="multiple" style="width: 3rem" :exportable="false"></Column>
                <Column field="name" header="Name" sortable style="min-width: 12rem"></Column>
                <Column field="value" header="Token" sortable style="min-width: 20rem">
                    <template #body="slotProps">
                        <span style="display: flex; align-items: center">
                            <span style="font-family: monospace; letter-spacing: 1px">
                                {{ tokenVisibility[slotProps.data.id] ? slotProps.data.value : '*'.repeat(slotProps.data.value.length) }}
                            </span>
                            <Button
                                :icon="tokenVisibility[slotProps.data.id] ? 'pi pi-eye-slash' : 'pi pi-eye'"
                                text
                                rounded
                                class="ml-2"
                                @click="toggleTokenVisibility(slotProps.data.id)"
                                :aria-label="tokenVisibility[slotProps.data.id] ? 'Hide token' : 'Show token'"
                            />
                        </span>
                    </template>
                </Column>
                <Column field="permissions" header="Permissions" style="min-width: 14rem">
                    <template #body="slotProps">
                        <span>
                            {{ slotProps.data.permissions && slotProps.data.permissions.length ? slotProps.data.permissions.map((p) => p.charAt(0).toUpperCase() + p.slice(1)).join(', ') : 'None' }}
                        </span>
                    </template>
                </Column>
                <Column field="description" header="Description" sortable style="min-width: 16rem"></Column>
                <Column field="createdAt" header="Created At" sortable style="min-width: 16rem">
                    <template #body="slotProps">
                        {{ new Date(slotProps.data.createdAt).toLocaleString() }}
                    </template>
                </Column>
                <Column :exportable="false" style="min-width: 12rem">
                    <template #body="slotProps">
                        <Button icon="pi pi-pencil" outlined rounded class="mr-2" @click="editToken(slotProps.data)" :disabled="!permissions.canEdit" />
                        <Button icon="pi pi-trash" outlined rounded severity="danger" @click="confirmDeleteToken(slotProps.data)" :disabled="!permissions.canDelete" />
                    </template>
                </Column>
            </DataTable>
        </div>

        <Dialog v-model:visible="tokenDialog" :style="{ width: '450px' }" header="Token Details" :modal="true">
            <div class="flex flex-col gap-6">
                <div>
                    <label for="name" class="block font-bold mb-3">Name</label>
                    <InputText id="name" v-model.trim="token.name" required="true" autofocus :invalid="submitted && !token.name" fluid />
                    <small v-if="submitted && !token.name" class="text-red-500">Name is required.</small>
                </div>
                <div>
                    <label for="description" class="block font-bold mb-3">Description</label>
                    <Textarea id="description" v-model="token.description" rows="3" cols="20" fluid />
                </div>
                <div>
                    <label class="block font-bold mb-3">Permissions</label>
                    <div class="flex flex-col gap-2">
                        <div>
                            <Checkbox inputId="perm-create" v-model="token.permissions" value="create" />
                            <label for="perm-create" class="ml-2">Create</label>
                        </div>
                        <div>
                            <Checkbox inputId="perm-edit" v-model="token.permissions" value="edit" />
                            <label for="perm-edit" class="ml-2">Edit</label>
                        </div>
                        <div>
                            <Checkbox inputId="perm-delete" v-model="token.permissions" value="delete" />
                            <label for="perm-delete" class="ml-2">Delete</label>
                        </div>
                        <div>
                            <Checkbox inputId="perm-export" v-model="token.permissions" value="export" />
                            <label for="perm-export" class="ml-2">Export</label>
                        </div>
                    </div>
                </div>
                <div v-if="token.value">
                    <label class="block font-bold mb-3">Token</label>
                    <span style="font-family: monospace; letter-spacing: 1px">
                        {{ token.value }}
                    </span>
                </div>
            </div>
            <template #footer>
                <Button label="Cancel" icon="pi pi-times" text @click="hideDialog" />
                <Button label="Save" icon="pi pi-check" @click="saveToken" :disabled="(!permissions.canEdit && token.id) || (!permissions.canCreate && !token.id)" />
            </template>
        </Dialog>

        <Dialog v-model:visible="deleteTokenDialog" :style="{ width: '450px' }" header="Confirm" :modal="true">
            <div class="flex items-center gap-4">
                <i class="pi pi-exclamation-triangle !text-3xl" />
                <span v-if="token"
                    >Are you sure you want to delete <b>{{ token.name }}</b
                    >?</span
                >
            </div>
            <template #footer>
                <Button label="No" icon="pi pi-times" text @click="deleteTokenDialog = false" />
                <Button label="Yes" icon="pi pi-check" @click="deleteToken" :disabled="!permissions.canDelete" />
            </template>
        </Dialog>

        <Dialog v-model:visible="deleteTokensDialog" :style="{ width: '450px' }" header="Confirm" :modal="true">
            <div class="flex items-center gap-4">
                <i class="pi pi-exclamation-triangle !text-3xl" />
                <span>Are you sure you want to delete the selected tokens?</span>
            </div>
            <template #footer>
                <Button label="No" icon="pi pi-times" text @click="deleteTokensDialog = false" />
                <Button label="Yes" icon="pi pi-check" text @click="deleteSelectedTokens" :disabled="!permissions.canDelete" />
            </template>
        </Dialog>
    </div>
</template>
