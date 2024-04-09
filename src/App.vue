<script lang="ts" setup>
  import { Close } from '@element-plus/icons-vue'
  import type { FormInstance, FormRules } from 'element-plus'
  import { ref, computed } from 'vue'

  interface RuleForm {
    files: string[]
    extension: string
    extensionOther: string
    from: string
    to: string
  }

  const isDevelopment = import.meta.env.DEV
  const package_version = 'v' + PACKAGE_VERSION

  const ruleFormRef = ref<FormInstance>()
  const ruleForm = ref<RuleForm>({
    files: [],
    extension: 'bms',
    extensionOther: '',
    from: '',
    to: '',
  })
  const rules = computed<FormRules<RuleForm>>(() => ({
    files: [{ required: true, message: '請選擇檔案資料夾路徑' }],
    extension: [
      { required: true, message: '請選擇檔案格式' },
      {
        validator: (rule, value, callback) => {
          if (value === 'other') callback(ruleForm.value.extensionOther === '' ? '請輸入檔案格式' : undefined)
          else callback()
        },
        message: '請輸入檔案格式',
        trigger: 'blur',
      },
    ],
    from: [{ required: !isRemoveBannedText.value, message: '請輸入搜尋內容', trigger: 'change' }],
  }))

  const enabledRecursive = ref(false)
  const isRemoveBannedText = ref(false)
  const enabledDryRun = ref(true)
  const options = [
    {
      label: 'bms',
      value: 'bms',
    },
    {
      label: 'bml',
      value: 'bml',
    },
    {
      label: 'bme',
      value: 'bme',
    },

    {
      label: '其他',
      value: 'other',
    },
  ]

  const isLoading = ref(false)

  const dialogVisible = ref(false)
  const dialogProps = ref({
    icon: '',
    title: '',
    subTitle: '',
    buttonText: '',
  })

  const handlePathSelection = async () => {
    try {
      ruleForm.value.files = await window?.electronAPI.openFile()
      await validateFiles()
    } catch (e) {}
  }

  const handleItemRemove = async (index) => {
    ruleForm.value.files.splice(index, 1)
    await validateFiles()
  }

  const validateFiles = async () => {
    try {
      await ruleFormRef.value?.validateField('files')
    } catch (e) {}
  }

  const handleSubmit = (formEl: FormInstance | undefined) => {
    if (!formEl) return
    formEl.validate((valid, fields) => {
      if (valid) {
        handleRequest()
      } else {
        console.log('error submit!', fields)
      }
    })
  }

  const resetForm = (formEl: FormInstance | undefined) => {
    if (!formEl) return
    formEl.resetFields()
  }

  const handleRequest = async () => {
    const files = ruleForm.value.files.map((file) => {
      if (enabledRecursive.value) file += '\\**'
      const extension = ruleForm.value.extension === 'other' ? ruleForm.value.extensionOther : ruleForm.value.extension
      file += `\\*.${extension.toLocaleLowerCase()}`
      return file
    })
    let from = new RegExp('')
    let to = ''

    if (isRemoveBannedText.value) {
      from = /#(TOTAL|Beatmania|Bemani)/g
    } else {
      from = new RegExp(ruleForm.value.from, 'g')
      to = ruleForm.value.to
    }

    const payload = {
      files,
      from,
      to,
      glob: {
        windowsPathsNoEscape: true, //To fix paths on Windows OS when path.join() is used to create paths
      },
    }
    if (isDevelopment) payload.dry = enabledDryRun.value
    try {
      isLoading.value = true
      const res = await window?.electronAPI.submit(payload)
      handleResponse(res)
    } catch (e) {
      console.log('error :>> ', e)
    } finally {
      isLoading.value = false
    }
  }

  const handleResponse = ([success, result]) => {
    if (success) {
      const matchCount = result.length
      const changedFilesCount = result.filter((result) => result.hasChanged).map((result) => result.file).length
      const props = {
        icon: 'success',
        title: '成功',
        subTitle: `匹配 ${matchCount} 份檔案，成功修改 ${changedFilesCount} 份檔案`,
        buttonText: '知道了',
      }
      showDialog(props)
    } else {
      let errorMessage = result.message
      if (result.message.includes('No files match the pattern')) errorMessage = '找不到匹配的檔案'
      const props = {
        icon: 'error',
        title: '發生錯誤',
        subTitle: errorMessage,
        buttonText: '把牛排變成鱒魚，知道了',
      }
      showDialog(props)
    }
  }

  const showDialog = (props) => {
    dialogProps.value = props
    dialogVisible.value = true
  }
</script>
<template>
  <el-form
    ref="ruleFormRef"
    style="max-width: 600px"
    :model="ruleForm"
    :rules="rules"
    :validate-on-rule-change="false"
    label-width="auto"
    label-position="left"
    require-asterisk-position="right"
    class="demo-ruleForm"
  >
    <el-form-item label="檔案資料夾路徑" prop="files">
      <el-button type="success" plain @click="handlePathSelection">選擇路徑</el-button>
      <div v-if="ruleForm.files.length" class="path-list">
        <div v-for="(path, index) in ruleForm.files" class="item">
          <div>{{ path }}</div>
          <el-button type="danger" size="small" :icon="Close" @click="handleItemRemove(index)" plain circle />
        </div>
      </div>
    </el-form-item>
    <el-form-item label="包含子資料夾" class="label-verticle-center">
      <el-switch v-model="enabledRecursive" inline-prompt size="large" active-text="是" inactive-text="否" />
    </el-form-item>

    <el-divider />

    <el-form-item label="檔案格式" prop="extension">
      <el-radio-group v-model="ruleForm.extension" class="radio-group">
        <el-radio v-for="item in options" :value="item.value" border>{{ item.label }}</el-radio>
      </el-radio-group>
      <template v-if="ruleForm.extension === 'other'">
        <br />
        <el-input v-model="ruleForm.extensionOther" placeholder="輸入檔案格式" style="margin-top: 18px" clearable />
      </template>
    </el-form-item>
    <el-form-item label="移除禁字" class="label-verticle-center">
      <el-switch v-model="isRemoveBannedText" inline-prompt size="large" active-text="是" inactive-text="否" />
    </el-form-item>

    <div v-show="!isRemoveBannedText">
      <el-form-item label="搜尋內容" prop="from">
        <el-input v-model="ruleForm.from" placeholder="輸入搜尋內容" clearable />
      </el-form-item>
      <el-form-item label="取代為" prop="to">
        <el-input v-model="ruleForm.to" clearable />
      </el-form-item>
    </div>

    <div style="text-align: center; margin-top: 40px">
      <el-checkbox
        v-if="isDevelopment"
        v-model="enabledDryRun"
        label="模擬結果"
        size="large"
        style="margin-right: 15px"
      />
      <el-button type="primary" size="large" :loading="isLoading" round @click="handleSubmit(ruleFormRef)">
        {{ isLoading ? '處理中...' : '開始取代' }}
      </el-button>
      <el-button @click="resetForm(ruleFormRef)">重置</el-button>
    </div>
  </el-form>
  <el-dialog v-model="dialogVisible" width="320" :show-close="false" align-center>
    <el-result :icon="dialogProps.icon" :title="dialogProps.title" :sub-title="dialogProps.subTitle">
      <template #extra>
        <el-button type="primary" @click="dialogVisible = false">{{ dialogProps.buttonText }}</el-button>
      </template>
    </el-result>
  </el-dialog>
  <div class="ver">{{ package_version }}</div>
</template>

<style lang="scss" scoped>
  .path-list {
    width: 100%;
    margin-top: 10px;
    .item {
      display: flex;
      justify-content: space-between;
      align-items: center;

      & + .item {
        margin-top: 8px;
      }
    }
  }

  :deep(.label-verticle-center) {
    .el-form-item__label {
      align-self: center;
    }
  }

  :deep(.radio-group) {
    .el-radio {
      &.is-bordered {
        margin-right: 20px;
      }
    }
  }

  .ver {
    position: fixed;
    right: 5px;
    bottom: 5px;
    opacity: 0.3;
  }
</style>
