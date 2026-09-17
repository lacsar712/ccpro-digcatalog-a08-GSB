<template>
  <div>
    <div class="toolbar">
      <div>
        <h2 class="page-title">探方摄影台账</h2>
        <p class="page-sub">按探方登记拍摄编号、方向、内容与关联登记号</p>
      </div>
      <button class="btn" @click="openCreate">新增台账</button>
    </div>

    <div class="card">
      <div class="filters">
        <label>
          探方筛选
          <select v-model="filterUnitId" @change="load">
            <option value="">全部探方</option>
            <option v-for="u in units" :key="u.id" :value="String(u.id)">
              {{ u.site?.name || '' }} / {{ u.code }}
            </option>
          </select>
        </label>
        <label>
          拍摄日期
          <input v-model="filterDate" type="date" @change="load" />
        </label>
        <button v-if="filterUnitId || filterDate" class="btn secondary small clear-btn" @click="clearFilters">
          清除筛选
        </button>
      </div>

      <table class="table">
        <thead>
          <tr>
            <th>照片编号</th>
            <th>探方</th>
            <th>拍摄日期</th>
            <th>拍摄方向</th>
            <th>拍摄内容</th>
            <th>关联登记号</th>
            <th>文件引用</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in list" :key="item.id">
            <td>{{ item.photoNo }}</td>
            <td>{{ item.unit?.code || '-' }}</td>
            <td>{{ formatDate(item.shotAt) }}</td>
            <td>{{ item.direction || '-' }}</td>
            <td>{{ item.subject || '-' }}</td>
            <td>
              <span v-if="item.linkedFind" class="tag">{{ item.linkedFind.registerNo }}</span>
              <span v-else>-</span>
            </td>
            <td class="file-ref">
              <a v-if="isLink(item.fileRef)" :href="item.fileRef" target="_blank" rel="noopener">{{ item.fileRef }}</a>
              <span v-else>{{ item.fileRef || '-' }}</span>
            </td>
            <td>
              <button class="btn secondary small" @click="openEdit(item)">编辑</button>
              <button class="btn danger small" @click="remove(item)">删除</button>
            </td>
          </tr>
        </tbody>
      </table>
      <p v-if="!list.length" class="page-sub">暂无数据</p>
      <p v-if="error" class="error">{{ error }}</p>
    </div>

    <div v-if="showModal" class="modal-mask" @click.self="showModal = false">
      <div class="modal">
        <h3>{{ form.id ? '编辑台账' : '新增台账' }}</h3>
        <div class="form-grid">
          <label>
            所属探方
            <select v-model.number="form.unitId" @change="onFormUnitChange">
              <option :value="0" disabled>请选择</option>
              <option v-for="u in units" :key="u.id" :value="u.id">
                {{ u.site?.name || '' }} / {{ u.code }}
              </option>
            </select>
          </label>
          <label>
            照片编号
            <input v-model="form.photoNo" placeholder="如 T1-20240312-001" />
          </label>
          <label>
            拍摄日期
            <input v-model="form.shotAt" type="date" />
          </label>
          <label>
            拍摄方向
            <input v-model="form.direction" placeholder="如 由北向南 / 垂直俯拍" />
          </label>
          <label class="full">
            拍摄内容
            <textarea v-model="form.subject" placeholder="拍摄对象与场景说明" />
          </label>
          <label>
            关联登记号(可选)
            <select v-model="form.linkedFindId">
              <option :value="null">不关联</option>
              <option v-for="f in unitFinds" :key="f.id" :value="f.id">{{ f.registerNo }}</option>
            </select>
          </label>
          <label>
            文件引用
            <input v-model="form.fileRef" placeholder="路径或外链,如 photos/2024/T1/DSC_0001.jpg" />
          </label>
        </div>
        <p v-if="formError" class="error">{{ formError }}</p>
        <div class="modal-actions">
          <button class="btn secondary" @click="showModal = false">取消</button>
          <button class="btn" @click="save">保存</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, reactive, ref } from 'vue'
import { useRoute } from 'vue-router'
import api from '../api/http'

const route = useRoute()

const list = ref([])
const units = ref([])
const unitFinds = ref([])
const filterUnitId = ref(route.query.unitId ? String(route.query.unitId) : '')
const filterDate = ref('')
const error = ref('')
const formError = ref('')
const showModal = ref(false)

const form = reactive({
  id: null,
  unitId: 0,
  photoNo: '',
  shotAt: '',
  direction: '',
  subject: '',
  linkedFindId: null,
  fileRef: ''
})

function formatDate(v) {
  if (!v) return '-'
  return String(v).slice(0, 10)
}

function isLink(v) {
  return typeof v === 'string' && /^https?:\/\//.test(v)
}

async function loadUnits() {
  const { data } = await api.get('/units')
  units.value = data
}

async function loadUnitFinds(unitId) {
  if (!unitId) {
    unitFinds.value = []
    return
  }
  const { data } = await api.get('/finds', { params: { unitId } })
  unitFinds.value = data
}

async function load() {
  error.value = ''
  try {
    const params = {}
    if (filterUnitId.value) params.unitId = filterUnitId.value
    if (filterDate.value) params.date = filterDate.value
    const { data } = await api.get('/photologs', { params })
    list.value = data
  } catch (e) {
    error.value = e.response?.data?.error || '加载失败'
  }
}

function clearFilters() {
  filterUnitId.value = ''
  filterDate.value = ''
  load()
}

async function onFormUnitChange() {
  form.linkedFindId = null
  await loadUnitFinds(form.unitId)
}

async function openCreate() {
  Object.assign(form, {
    id: null,
    unitId: filterUnitId.value ? Number(filterUnitId.value) : units.value[0]?.id || 0,
    photoNo: '',
    shotAt: '',
    direction: '',
    subject: '',
    linkedFindId: null,
    fileRef: ''
  })
  formError.value = ''
  showModal.value = true
  await loadUnitFinds(form.unitId)
}

async function openEdit(item) {
  Object.assign(form, {
    id: item.id,
    unitId: item.unitId,
    photoNo: item.photoNo,
    shotAt: formatDate(item.shotAt) === '-' ? '' : formatDate(item.shotAt),
    direction: item.direction || '',
    subject: item.subject || '',
    linkedFindId: item.linkedFindId ?? null,
    fileRef: item.fileRef || ''
  })
  formError.value = ''
  showModal.value = true
  await loadUnitFinds(form.unitId)
}

async function save() {
  formError.value = ''
  try {
    const payload = {
      unitId: form.unitId,
      photoNo: form.photoNo,
      shotAt: form.shotAt || null,
      direction: form.direction,
      subject: form.subject,
      linkedFindId: form.linkedFindId || null,
      fileRef: form.fileRef
    }
    if (form.id) {
      await api.put(`/photologs/${form.id}`, payload)
    } else {
      await api.post('/photologs', payload)
    }
    showModal.value = false
    await load()
  } catch (e) {
    formError.value = e.response?.data?.error || '保存失败'
  }
}

async function remove(item) {
  if (!confirm(`确认删除台账「${item.photoNo}」?`)) return
  try {
    await api.delete(`/photologs/${item.id}`)
    await load()
  } catch (e) {
    alert(e.response?.data?.error || '删除失败')
  }
}

onMounted(async () => {
  await loadUnits()
  await load()
})
</script>

<style scoped>
.filters {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
  flex-wrap: wrap;
  align-items: flex-end;
}

.filters label {
  min-width: 200px;
}

.clear-btn {
  margin-bottom: 0.15rem;
}

.file-ref {
  max-width: 220px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
</style>
