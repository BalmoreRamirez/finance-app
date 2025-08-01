<template>
  <Toast />
  <ConfirmDialog />

  <div class="space-y-6">
    <div class="flex justify-between items-center">
      <h1 class="text-3xl font-bold text-gray-800">Gestión de Cuentas de Balance</h1>
      <Button label="Nueva Cuenta" icon="pi pi-plus" @click="openCreateModal" />
    </div>

    <div class="bg-white rounded-lg shadow-md overflow-hidden">
      <div class="overflow-x-auto">
        <table class="min-w-full divide-y divide-gray-200">
          <thead class="bg-gray-50">
            <tr>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Nombre</th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Tipo</th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Saldo</th>
              <th class="px-6 py-3 text-center text-xs font-medium text-gray-500 uppercase">Acciones</th>
            </tr>
          </thead>
          <tbody class="bg-white divide-y divide-gray-200">
            <tr v-for="(cuenta, idx) in cuentasDeBalance" :key="cuenta.id">
              <td class="px-6 py-4 text-sm font-medium text-gray-900">{{ cuenta.name }}</td>
              <td class="px-6 py-4 text-sm">
                <Tag :value="getTipoLabel( cuenta.account_type.name)" :severity="getTipoTagSeverity( cuenta.account_type.name)" />
              </td>
              <td class="px-6 py-4 text-sm font-semibold text-gray-700">
                {{ formatCurrency(cuenta.balance, cuenta.currency) }}
              </td>
              <td class="px-6 py-4 text-center space-x-2">
                <Button icon="pi pi-pencil" class="p-button-rounded p-button-info p-button-text" @click="openEditModal(cuenta)" />
                <Button icon="pi pi-trash" class="p-button-rounded p-button-danger p-button-text" @click="handleDeleteCuenta(cuenta.id)" />
              </td>
            </tr>
            <tr v-if="cuentasDeBalance.length === 0">
              <td colspan="4" class="px-6 py-4 text-center text-gray-500">No hay cuentas de activo o pasivo registradas.</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>

  <CuentaModal
    v-model:visible="showModal"
    :is-edit-mode="isEditMode"
    :cuenta-data="cuentaToEdit"
    :tipos-cuenta="tiposCuentaDeBalance"
    @save="handleSaveCuenta"
  />
</template>

<script setup>
import { ref, computed, onMounted } from "vue";
import { useFinanceStore } from "../../../../stores/financeStore.js";
import { storeToRefs } from "pinia";
import { useConfirm } from "primevue/useconfirm";
import { useToast } from "primevue/usetoast";
import Button from "primevue/button";
import ConfirmDialog from "primevue/confirmdialog";
import Toast from "primevue/toast";
import CuentaModal from "../components/CuentaModal.vue";
import Tag from "primevue/tag";

// Store setup
const store = useFinanceStore();
const confirm = useConfirm();
const toast = useToast();

const {
  fetchAccounts,
  fetchAccountTypes,
  addCuenta,
  updateCuenta,
  deleteCuenta,
} = store;
const { accounts, accountTypes } = storeToRefs(store);

// State
const showModal = ref(false);
const isEditMode = ref(false);
const cuentaToEdit = ref(null);

// Lifecycle
onMounted(() => {
  if (accounts.value.length === 0) fetchAccounts();
  if (accountTypes.value.length === 0) fetchAccountTypes();
});

// Computed
const cuentasDeBalance = computed(() => accounts.value);
const tiposCuentaDeBalance = computed(() => accountTypes.value);

// Helpers
const formatCurrency = (value, currencyCode = "USD") =>
  new Intl.NumberFormat("es-MX", { style: "currency", currency: currencyCode }).format(value);

const getTipoLabel = (tipoValue) => {
  if (!tipoValue) return "No Asignado";
  const tipo = accountTypes.value.find(
    (t) => t.nombre.toLowerCase() === tipoValue.toLowerCase()
  );
  return tipo ? tipo.nombre : tipoValue;
};

const getTipoTagSeverity = (tipo) => {
  switch (tipo) {
    case "activo":
      return "success";
    case "pasivo":
      return "warning";
    default:
      return "secondary";
  }
};

// Modal handlers
const openCreateModal = () => {
  isEditMode.value = false;
  cuentaToEdit.value = {
    name: "",
    account_type_id: null,
    balance: 0,
    currency: "USD",
  };
  showModal.value = true;
};

const openEditModal = (cuenta) => {
  isEditMode.value = true;
  let tipoCuentaObj = null;
  if (cuenta.account_type_id && typeof cuenta.account_type_id === "object") {
    tipoCuentaObj = accountTypes.value.find(tc => tc.value === cuenta.account_type_id.value) || cuenta.account_type_id;
  } else if (cuenta.account_type_id) {
    tipoCuentaObj = accountTypes.value.find(tc => tc.value === cuenta.account_type_id) || null;
  } else if (cuenta.accountTypeName) {
    tipoCuentaObj = accountTypes.value.find(tc => tc.nombre.toLowerCase() === cuenta.accountTypeName.toLowerCase()) || null;
  } else if (cuenta.tipo) {
    tipoCuentaObj = accountTypes.value.find(tc => tc.nombre.toLowerCase() === cuenta.tipo.toLowerCase()) || null;
  }
  cuentaToEdit.value = {
    accountId: cuenta.id,
    name: cuenta.name,
    account_type_id: tipoCuentaObj,
    balance: cuenta.balance ?? cuenta.saldo ?? 0,
    currency: cuenta.currency || cuenta.moneda || "USD",
  };
  showModal.value = true;
};

const handleSaveCuenta = async (cuentaData) => {
  let success = false;
  if (isEditMode.value) {
    success = await updateCuenta(cuentaData);
  } else {
    success = await addCuenta(cuentaData);
  }

  toast.add({
    severity: success ? "success" : "error",
    summary: success ? "Éxito" : "Error",
    detail: success
      ? `Cuenta ${isEditMode.value ? "actualizada" : "creada"} correctamente.`
      : `No se pudo ${isEditMode.value ? "actualizar" : "crear"} la cuenta.`,
    life: 3000,
  });
  showModal.value = false;
};

const handleDeleteCuenta = async (cuentaId) => {
  confirm.require({
    message: "¿Estás seguro de eliminar esta cuenta? Esta acción no se puede deshacer.",
    header: "Confirmar Eliminación",
    icon: "pi pi-exclamation-triangle",
    acceptClass: "p-button-danger",
    acceptLabel: "Sí, eliminar",
    rejectLabel: "Cancelar",
    accept: async () => {
      const success = await deleteCuenta(cuentaId);
      toast.add({
        severity: success ? "success" : "error",
        summary: success ? "Éxito" : "Error",
        detail: success ? "Cuenta eliminada." : "No se pudo eliminar la cuenta.",
        life: 3000,
      });
    },
  });
};
</script>