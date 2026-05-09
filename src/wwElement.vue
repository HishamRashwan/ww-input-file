<template>
    <div
        class="ww-file-upload"
        :class="{
            'ww-file-upload--dragging': isDragging && !isDisabled && !isReadonly,
            'ww-file-upload--disabled': isDisabled,
            'ww-file-upload--readonly': isReadonly,
            'ww-file-upload--has-files': hasFiles,
        }"
        @dragover="handleDragOver"
        @dragleave="handleDragLeave"
        @drop="handleDrop"
        role="region"
        aria-label="File upload area"
    >
        <!-- Main upload area -->
        <div ref="dropzoneEl" class="ww-file-upload__dropzone" @click="openFileExplorer" @mousemove="handleMouseMove">
            <div
                v-if="isDragging && !isDisabled && !isReadonly && enableCircleAnimation"
                ref="circleEl"
                class="ww-file-upload__hover-circle"
                :style="{
                    left: `${mouseX}px`,
                    top: `${mouseY}px`,
                    backgroundColor: circleColor,
                    opacity: circleOpacity,
                    width: circleSize,
                    height: circleSize,
                }"
            ></div>

            <div class="ww-file-upload__content" :class="[`ww-file-upload__content--${uploadIconPosition}`]">
                <div v-if="showUploadIcon" class="ww-file-upload__icon" v-html="iconHTML" />
                <div class="ww-file-upload__text">
                    <div class="ww-file-upload__label" :style="labelMessageStyle">{{ labelMessage }}</div>
                    <div
                        class="ww-file-upload__info ww-file-upload__extensions-message"
                        v-if="extensionsMessage"
                        :style="extensionsMessageStyle"
                    >
                        {{ extensionsMessage }}
                    </div>
                    <div
                        class="ww-file-upload__info ww-file-upload__max-file-message"
                        v-if="maxFileMessage"
                        :style="maxFileMessageStyle"
                    >
                        {{ maxFileMessage }}
                    </div>
                </div>
            </div>
        </div>

        <!-- File list -->
        <FileList
            v-if="hasFiles"
            :files="fileList"
            :status="status"
            :type="type"
            :can-reorder="reorder"
            :is-readonly="isReadonly"
            :is-disabled="isDisabled"
            @remove="removeFile"
            @reorder="reorderFiles"
        />

        <!-- Hidden file input -->
        <input
            ref="fileInput"
            type="file"
            class="ww-file-upload__input"
            :multiple="type === 'multi'"
            :accept="acceptedFileTypes"
            :required="required && !hasFiles"
            :disabled="isDisabled || isReadonly"
            :aria-label="labelMessage"
            @change="handleFileSelection"
        />
    </div>
</template>

<script>
import { ref, computed, watch, provide, inject, onBeforeUnmount } from 'vue';
import FileList from './components/FileList.vue';
import { getFileExtension, validateFile } from './utils/fileValidation';
import { fileToBase64, fileToBinary } from './utils/fileProcessing';
import { useDragAnimation } from './composables/useDragAnimation';

/* wwEditor:start */
import useParentSelection from './editor/useParentSelection';
/* wwEditor:end */

export default {
    components: {
        FileList,
    },
    props: {
        content: { type: Object, required: true },
        /* wwEditor:start */
        wwFrontState: { type: Object, required: true },
        wwEditorState: { type: Object, required: true },
        parentSelection: { type: Object, default: () => ({ allow: false, texts: {} }) },
        /* wwEditor:end */
        uid: { type: String, required: true },
        wwElementState: { type: Object, required: true },
    },
    emits: ['trigger-event', 'add-state', 'remove-state'],
    setup(props, { emit }) {

        const isEditing = computed(() => {
            /* wwEditor:start */
            return props.wwEditorState?.isEditing;
            /* wwEditor:end */
            // eslint-disable-next-line no-unreachable
            return false;
        });

        /* wwEditor:start */
        const { selectParentElement } = useParentSelection(props, emit);
        /* wwEditor:end */

        const { getIcon } = wwLib.useIcons();

        const fileInput = ref(null);
        const dropzoneEl = ref(null);
        const circleEl = ref(null);
        const iconText = ref(null);
        const isProcessing = ref(false);
        const isUploading = ref(false);
        const activeUploads = ref({});
        const activeUploadProgress = ref({});

        const extensionsMessage = computed(() => props.content?.extensionsMessage || null);
        const maxFileMessage = computed(() => props.content?.maxFileMessage || null);
        const labelMessage = computed(() => props.content?.labelMessage || null);

        const type = computed(() => props.content?.type || 'single');
        const reorder = computed(() => props.content?.reorder || false);
        const drop = computed(() => props.content?.drop !== false);
        const maxFileSize = computed(() => props.content?.maxFileSize || 10);
        const minFileSize = computed(() => props.content?.minFileSize || 0);
        const maxTotalFileSize = computed(() => props.content?.maxTotalFileSize || 50);
        const maxFiles = computed(() => props.content?.maxFiles || 10);
        const required = computed(() => props.content?.required || false);
        const extensions = computed(() => props.content?.extensions || 'any');
        const customExtensions = computed(() => props.content?.customExtensions || '');
        const autoUpload = computed(() => props.content?.autoUpload !== false);
        const exposeBase64 = computed(() => props.content?.exposeBase64 || false);
        const exposeBinary = computed(() => props.content?.exposeBinary || false);
        const showUploadIcon = computed(() => props.content?.showUploadIcon !== false);
        const uploadIcon = computed(() => props.content?.uploadIcon || 'upload');
        const uploadIconColor = computed(() => props.content?.uploadIconColor || '#666666');
        const uploadIconPosition = computed(() => props.content?.uploadIconPosition || 'top');
        const uploadIconSize = computed(() => props.content?.uploadIconSize || '24px');
        const uploadIconMargin = computed(() => props.content?.uploadIconMargin || '8px');
        const enableCircleAnimation = computed(() => props.content?.enableCircleAnimation !== false);
        const circleSize = computed(() => props.content?.circleSize || '80px');
        const circleColor = computed(() => props.content?.circleColor || safeProgressBarColor.value);
        const circleOpacity = computed(() =>
            props.content?.circleOpacity !== undefined ? props.content.circleOpacity : 0.5
        );
        const animationSpeed = computed(() =>
            props.content?.animationSpeed !== undefined ? props.content.animationSpeed : 0.5
        );

        const safeDropzoneBorderWidth = computed(() => props.content?.dropzoneBorderWidth || '2px');
        const safeDropzoneBorderStyle = computed(() => props.content?.dropzoneBorderStyle || 'dashed');
        const safeDropzoneBorderColor = computed(() => props.content?.dropzoneBorderColor || '#CCCCCC');
        const safeDropzoneBorderRadius = computed(() => props.content?.dropzoneBorderRadius || '8px');
        const safeDropzonePadding = computed(() => props.content?.dropzonePadding || '20px');
        const safeDropzoneMinHeight = computed(() => props.content?.dropzoneMinHeight || '120px');
        const safeDropzoneBackground = computed(() => props.content?.dropzoneBackground || 'rgba(0, 0, 0, 0)');
        const safeDropzoneBackgroundHover = computed(
            () => props.content?.dropzoneBackgroundHover || 'rgba(0, 0, 0, 0.01)'
        );
        const safeLabelFontSize = computed(() => props.content?.labelFontSize || '16px');
        const safeLabelFontFamily = computed(() => props.content?.labelFontFamily || 'inherit');
        const safeLabelFontWeight = computed(() => props.content?.labelFontWeight || 'normal');
        const safeLabelColor = computed(() => props.content?.labelColor || '#333333');
        const safeFileDetailsFontSize = computed(() => props.content?.fileDetailsFontSize || '12px');
        const safeFileDetailsColor = computed(() => props.content?.fileDetailsColor || '#888888');
        const safeProgressBarColor = computed(() => props.content?.progressBarColor || '#EEEEEE');
        const safeLabelMarginBottom = computed(() => {
            const labelMargin = props.content?.labelMargin || '0 0 4px 0';
            return labelMargin.split(' ')[2] || '4px';
        });
        const safeProgressBarBackground = computed(() => {
            const color = safeProgressBarColor.value;
            if (!color) return 'rgba(85, 85, 85, 0.05)';

            try {
                const r = parseInt(color.slice(1, 3), 16) || 85;
                const g = parseInt(color.slice(3, 5), 16) || 85;
                const b = parseInt(color.slice(5, 7), 16) || 85;
                return `rgba(${r}, ${g}, ${b}, 0.05)`;
            } catch (e) {
                return 'rgba(85, 85, 85, 0.05)';
            }
        });

        const safeDropzoneBackgroundDragging = computed(
            () => props.content?.dropzoneBackgroundDragging || 'rgba(0, 0, 0, 0.05)'
        );

        // Extensions message styling
        const extensionsMessageStyle = computed(() => ({
            fontFamily: props.content?.extensionsMessageFontFamily || 'inherit',
            fontSize: props.content?.extensionsMessageFontSize || '12px',
            fontWeight: props.content?.extensionsMessageFontWeight || 'normal',
            color: props.content?.extensionsMessageColor || '#888888',
            margin: props.content?.extensionsMessageMargin || '0 0 4px 0',
        }));

        // Max file message styling
        const maxFileMessageStyle = computed(() => ({
            fontFamily: props.content?.maxFileMessageFontFamily || 'inherit',
            fontSize: props.content?.maxFileMessageFontSize || '12px',
            fontWeight: props.content?.maxFileMessageFontWeight || 'normal',
            color: props.content?.maxFileMessageColor || '#888888',
            margin: props.content?.maxFileMessageMargin || '0 0 4px 0',
        }));

        // Label message styling
        const labelMessageStyle = computed(() => ({
            fontFamily: props.content?.labelFontFamily || 'inherit',
            fontSize: props.content?.labelFontSize || '16px',
            fontWeight: props.content?.labelFontWeight || 'normal',
            color: props.content?.labelColor || '#333333',
            margin: props.content?.labelMargin || '0 0 4px 0',
        }));

        const isDisabled = computed(() => props.wwElementState.props.disabled || false);
        const isReadonly = computed(() => {
            /* wwEditor:start */
            if (props.wwEditorState?.isSelected) {
                return props.wwElementState.states.includes('readonly');
            }
            /* wwEditor:end */
            return props.wwElementState.props.readonly === undefined
                ? props.content?.readonly || false
                : props.wwElementState.props.readonly;
        });

        const { value: files, setValue: setFiles } = wwLib.wwVariable.useComponentVariable({
            uid: props.uid,
            name: 'value',
            defaultValue: [],
            type: 'file',
            componentType: 'element',
        });

        const { value: status, setValue: setStatus } = wwLib.wwVariable.useComponentVariable({
            uid: props.uid,
            name: 'status',
            defaultValue: {},
            type: 'any',
        });

        const useForm = inject('_wwForm:useForm', () => {});
        const fieldName = computed(() => props.content.fieldName);
        const validation = computed(() => props.content.validation);
        const customValidation = computed(() => props.content.customValidation);

        useForm(
            files,
            { fieldName, validation, customValidation, required },
            { elementState: props.wwElementState, emit, sidepanelFormPath: 'form', setValue: setFiles }
        );

        const fileList = computed(() => (Array.isArray(files.value) ? files.value : []));
        const hasFiles = computed(() => fileList.value.length > 0);
        const hasSupabaseConfig = computed(
            () => props.content?.supabaseUrl && props.content?.supabaseAnonKey && props.content?.supabaseBucket
        );

        watch([status, fileList], ([newStatus, newFiles]) => {
            if (newStatus && typeof newStatus === 'object') {
                const fileKeys = newFiles.map(file => file.id || file.name);
                const updatedStatus = Object.fromEntries(
                    Object.entries(newStatus).filter(([key]) => fileKeys.includes(key))
                );

                if (Object.keys(updatedStatus).length !== Object.keys(newStatus).length) {
                    setStatus(updatedStatus);
                }
            }
        });

        const getFileStatus = file => {
            const statusKey = file?.id || file?.name;
            if (!status.value || !statusKey || !status.value[statusKey]) {
                return {
                    uploadProgress: 0,
                    isUploading: false,
                    isUploaded: false,
                };
            }

            return status.value[statusKey];
        };

        const acceptedFileTypes = computed(() => {
            switch (extensions.value) {
                case 'image':
                    return 'image/*';
                case 'video':
                    return 'video/*';
                case 'audio':
                    return 'audio/*';
                case 'pdf':
                    return '.pdf';
                case 'csv':
                    return '.csv';
                case 'excel':
                    return '.xls,.xlsx,.xlsm,.xlsb';
                case 'word':
                    return '.doc,.docx,.docm';
                case 'json':
                    return '.json';
                case 'custom':
                    return customExtensions.value;
                default:
                    return '';
            }
        });

        watch(
            () => uploadIcon.value,
            async icon => {
                if (icon && showUploadIcon.value) {
                    try {
                        iconText.value = await getIcon(icon);
                    } catch (error) {
                        iconText.value = null;
                    }
                } else {
                    iconText.value = null;
                }
            },
            { immediate: true }
        );

        const iconHTML = computed(() => {
            /* wwEditor:start */
            return (
                iconText.value ||
                '<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="feather feather-upload"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path><polyline points="17 8 12 3 7 8"></polyline><line x1="12" y1="3" x2="12" y2="15"></line></svg>'
            );
            /* wwEditor:end */
            return iconText.value;
        });

        const localData = ref({
            fileUpload: {
                value: computed(() => {
                    return fileList.value.map(file => {
                        const plainObject = {};
                        for (const key in file) {
                            if (Object.prototype.hasOwnProperty.call(file, key)) {
                                plainObject[key] = file[key];
                            }
                        }
                        plainObject.name = file.name;
                        plainObject.size = file.size;
                        plainObject.type = file.type;
                        plainObject.lastModified = file.lastModified;
                        plainObject.mimeType = file.mimeType;
                        plainObject.id = file.id;

                        if (file.base64) plainObject.base64 = file.base64;
                        if (file.binary) plainObject.binary = file.binary;

                        return plainObject;
                    });
                }),
                status: status,
                uploadProgress: computed(() => {
                    if (!fileList.value.length) return 0;
                    const average =
                        fileList.value.reduce((total, file) => {
                            const fileStatus = status.value?.[file.id || file.name] || {};
                            return total + (fileStatus.uploadProgress || 0);
                        }, 0) / fileList.value.length;
                    return Number(average.toFixed(1));
                }),
                isUploading,
                isUploaded: computed(() => Boolean(fileList.value.length) && fileList.value.every(file => file.isUploaded)),
            },
        });

        provide('_wwFileUpload', {
            files: fileList,
            status: status,
            acceptedTypes: acceptedFileTypes,
            isDisabled,
            isReadonly,
            isSingleMode: computed(() => type.value === 'single'),
            content: computed(() => props.content || {}),
        });

        // Drag and drop animation
        const {
            isDragging,
            mouseX,
            mouseY,
            targetX,
            targetY,
            isAnimating,
            handleDragOver: animationHandleDragOver,
            handleDragLeave: animationHandleDragLeave,
            handleDrop: animationHandleDrop,
            handleMouseMove: animationHandleMouseMove,
        } = useDragAnimation({
            dropzoneRef: dropzoneEl,
            circleRef: circleEl,
            isDisabled,
            isReadonly,
            dropEnabled: drop,
            circleOpacity,
            animationSpeed,
            isEditing,
        });

        // Methods
        const openFileExplorer = () => {
            if (!isDisabled.value && !isReadonly.value && !isEditing.value) {
                fileInput.value.click();
            }
        };

        const handleDrop = async event => {
            if (isDisabled.value || isReadonly.value || !drop.value || isEditing.value) return;
            event.preventDefault();
            event.stopPropagation();

            const animationHandled = animationHandleDrop(event);
            if (!animationHandled || isDisabled.value || isReadonly.value || !drop.value) return;

            const items = event.dataTransfer.files;
            if (!items.length) return;

            // Wait for the circle animation to complete before processing files
            setTimeout(async () => {
                await processFiles(items);
            }, 1050);
        };

        const handleFileSelection = async event => {
            const selectedFiles = event.target.files;
            if (!selectedFiles.length) return;

            await processFiles(selectedFiles);
            event.target.value = '';
        };

        const emitWorkflowEvent = (name, event) => {
            emit('trigger-event', { name, event });
        };

        const serializeFile = file => ({
            id: file.id,
            name: file.name,
            size: file.size,
            type: file.type,
            extension: file.extension || getFileExtension(file.name, file.type),
            path: file.path || null,
            fullPath: file.fullPath || null,
            publicUrl: file.publicUrl || null,
            bucket: file.bucket || null,
            isUploaded: Boolean(file.isUploaded),
        });

        const createWorkflowPayload = (sourceFiles = fileList.value) => {
            const value = sourceFiles.map(serializeFile);
            const uploadedFiles = value
                .filter(file => file.path)
                .map(file => ({
                    name: file.name,
                    path: file.path,
                    fullPath: file.fullPath,
                    publicUrl: file.publicUrl,
                }));
            const statusValue = status.value || {};
            const uploadProgress = value.length
                ? value.reduce((total, file) => {
                      const fileStatus = statusValue[file.id] || statusValue[file.name] || {};
                      return total + (fileStatus.uploadProgress || 0);
                  }, 0) / value.length
                : 0;

            return {
                value,
                status: statusValue,
                uploadProgress: Number(uploadProgress.toFixed(1)),
                isUploading: isUploading.value,
                isUploaded: Boolean(value.length) && value.every(file => file.isUploaded),
                files: uploadedFiles,
                filesJson: JSON.stringify(uploadedFiles),
                fileNames: uploadedFiles.map(file => file.name).join(', '),
                filePaths: uploadedFiles.map(file => file.path).join(', '),
            };
        };

        const emitChange = sourceFiles => {
            emitWorkflowEvent('change', createWorkflowPayload(sourceFiles));
        };

        const processFiles = async fileList => {
            isProcessing.value = true;
            const filesToProcess = Array.from(fileList);

            // Single mode handling
            if (type.value === 'single') {
                if (filesToProcess.length > 1) {
                    emitWorkflowEvent('error', {
                        code: 'SINGLE_MODE_MULTIPLE_FILES',
                        data: {
                            message: 'Multiple files provided in single file mode',
                            count: filesToProcess.length,
                            acceptedCount: 1,
                        },
                    });
                }

                filesToProcess.splice(1);
                setFiles([]);
            }

            let availableSlots = Infinity;
            if (type.value === 'multi' && maxFiles.value > 0) {
                availableSlots = maxFiles.value - files.value.length;
                if (availableSlots <= 0) {
                    emitWorkflowEvent('error', {
                        code: 'MAX_FILES_REACHED',
                        data: {
                            message: `Maximum number of files (${maxFiles.value}) reached`,
                            maxFiles: maxFiles.value,
                            currentCount: files.value.length,
                        },
                    });

                    wwLib.wwNotification.open({
                        text: { en: `Maximum number of files (${maxFiles.value}) reached` },
                        color: 'warning',
                    });

                    isProcessing.value = false;
                    return;
                } else if (filesToProcess.length > availableSlots) {
                    emitWorkflowEvent('error', {
                        code: 'TOO_MANY_FILES',
                        data: {
                            message: `Only ${availableSlots} more files can be added`,
                            providedCount: filesToProcess.length,
                            availableSlots: availableSlots,
                            maxFiles: maxFiles.value,
                            currentCount: files.value.length,
                        },
                    });
                }
            }

            // Process valid files
            const limitedFiles = filesToProcess.slice(0, availableSlots);
            const processedFiles = [];

            // Only calculate currentTotalSize for multi-file mode
            const currentTotalSize =
                type.value === 'multi' && Array.isArray(files.value)
                    ? files.value.reduce((sum, f) => sum + (f.size || 0), 0)
                    : 0;

            for (const file of limitedFiles) {
                const validationResult = validateFile(file, {
                    maxFileSize: maxFileSize.value,
                    minFileSize: minFileSize.value,
                    // Only apply maxTotalFileSize in multi-file mode
                    maxTotalFileSize: type.value === 'multi' ? maxTotalFileSize.value : undefined,
                    currentTotalSize: type.value === 'multi' ? currentTotalSize : 0,
                    acceptedTypes: acceptedFileTypes.value,
                });

                if (validationResult.valid) {
                    file.id = `file-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
                    file.mimeType = file.type;
                    if (exposeBase64.value) file.base64 = await fileToBase64(file);
                    if (exposeBinary.value) file.binary = await fileToBinary(file);
                    processedFiles.push(file);
                } else {
                    console.warn(`File validation failed: ${validationResult.reason}`);
                    emitWorkflowEvent('error', {
                        code: 'VALIDATION_ERROR',
                        data: {
                            message: validationResult.reason,
                            fileName: file.name,
                            fileSize: file.size,
                            fileType: file.type,
                            constraint: validationResult.constraint,
                        },
                    });

                    /* wwEditor:start */
                    wwLib.wwNotification.open({
                        text: { en: validationResult.reason },
                        color: 'error',
                    });
                    /* wwEditor:end */
                }
            }

            if (processedFiles.length > 0) {
                if (type.value === 'single') {
                    setFiles(processedFiles);
                    emitChange(processedFiles);
                    if (autoUpload.value) startUploading(processedFiles);
                    isProcessing.value = false;
                } else {
                    const currentFiles = [...files.value];
                    let newFiles = [...currentFiles];

                    const addNextFile = index => {
                        newFiles = [...newFiles, processedFiles[index]];
                        setFiles(newFiles);

                        if (index < processedFiles.length - 1) {
                            setTimeout(() => addNextFile(index + 1), 150);
                        } else {
                            emitChange(newFiles);
                            if (autoUpload.value) startUploading(newFiles);
                            isProcessing.value = false;
                        }
                    };

                    // Start adding files with a small initial delay
                    setTimeout(() => addNextFile(0), 50);
                    return;
                }
            }

            isProcessing.value = false;
        };

        const updateFileStatus = (file, patch) => {
            const key = file.id || file.name;
            setStatus({
                ...(status.value || {}),
                [key]: {
                    uploadProgress: 0,
                    isUploading: false,
                    isUploaded: false,
                    ...(status.value?.[key] || {}),
                    ...patch,
                    name: file.name,
                },
            });
        };

        const pruneStatus = nextFiles => {
            const allowedKeys = nextFiles.map(file => file.id || file.name);
            const nextStatus = Object.fromEntries(
                Object.entries(status.value || {}).filter(([key]) => allowedKeys.includes(key))
            );
            setStatus(nextStatus);
        };

        const startUploading = async (sourceFiles = fileList.value) => {
            if (isDisabled.value || isReadonly.value || isUploading.value) return;

            const pendingFiles = sourceFiles.filter(file => !file.isUploaded);
            if (!pendingFiles.length) return;

            if (!hasSupabaseConfig.value) {
                emitWorkflowEvent('error', {
                    code: 'SUPABASE_CONFIG_MISSING',
                    data: { message: 'Add Supabase URL, anon key, and bucket before uploading.' },
                });
                return;
            }

            isUploading.value = true;
            emitWorkflowEvent('upload-start', createWorkflowPayload());

            const bucket = String(props.content.supabaseBucket || '').trim();
            const successful = [];
            const failed = [];

            for (const file of pendingFiles) {
                try {
                    const path = createStoragePath(file);
                    const response = await uploadFileToSupabase(file, bucket, path);
                    const result = {
                        id: file.id,
                        name: file.name,
                        path: response.path || path,
                        fullPath: response.fullPath || `${bucket}/${response.path || path}`,
                        bucket,
                        size: file.size,
                        type: file.type,
                        publicUrl: props.content.returnPublicUrls ? getPublicUrl(bucket, response.path || path) : null,
                    };

                    Object.assign(file, result, { isUploaded: true });
                    updateFileStatus(file, {
                        uploadProgress: 100,
                        isUploading: false,
                        isUploaded: true,
                        path: result.path,
                        publicUrl: result.publicUrl,
                    });
                    successful.push(result);
                    emitWorkflowEvent('upload-success', { file: serializeFile(file), response: result });
                    emitChange();
                } catch (error) {
                    updateFileStatus(file, {
                        isUploading: false,
                        isUploaded: false,
                        error: error.message,
                    });
                    failed.push({ file: serializeFile(file), message: error.message, name: error.name });
                    emitWorkflowEvent('error', {
                        code: error.name || 'SUPABASE_UPLOAD_ERROR',
                        data: { message: error.message, fileName: file.name },
                    });
                    emitChange();
                }
            }

            isUploading.value = false;
            emitChange(sourceFiles);
            emitWorkflowEvent('complete', {
                ...createWorkflowPayload(sourceFiles),
                successful,
                failed,
            });
        };

        const uploadFileToSupabase = (file, bucket, path) =>
            new Promise((resolve, reject) => {
                const xhr = new XMLHttpRequest();
                const timeoutSeconds = toNonNegativeNumber(props.content.uploadTimeoutSeconds, 300);
                const url = createSupabaseObjectUrl(bucket, path);
                const key = file.id || file.name;

                activeUploads.value[key] = xhr;
                activeUploadProgress.value[key] = { loaded: 0, total: file.size, percentage: 0 };

                updateFileStatus(file, {
                    uploadProgress: 0,
                    isUploading: true,
                    isUploaded: false,
                    error: null,
                });
                emitChange();

                xhr.open('POST', url);
                xhr.timeout = timeoutSeconds ? timeoutSeconds * 1000 : 0;
                xhr.setRequestHeader('apikey', String(props.content.supabaseAnonKey || '').trim());
                xhr.setRequestHeader('Authorization', `Bearer ${String(props.content.supabaseAnonKey || '').trim()}`);
                xhr.setRequestHeader('Content-Type', file.type || 'application/octet-stream');
                xhr.setRequestHeader('Cache-Control', String(props.content.cacheControl || '3600'));
                xhr.setRequestHeader('x-upsert', props.content.upsert ? 'true' : 'false');

                xhr.upload.onprogress = event => {
                    if (!event.lengthComputable) return;

                    const percentage = Math.min(99, (event.loaded / event.total) * 100);
                    activeUploadProgress.value[key] = {
                        loaded: event.loaded,
                        total: event.total,
                        percentage,
                    };
                    updateFileStatus(file, {
                        uploadProgress: Number(Math.max(0.1, percentage).toFixed(1)),
                        isUploading: true,
                        bytesUploaded: event.loaded,
                        bytesTotal: event.total,
                    });
                    emitWorkflowEvent('upload-progress', {
                        file: serializeFile(file),
                        progress: status.value?.[key] || {},
                        status: status.value,
                    });
                    emitChange();
                };

                xhr.onload = () => {
                    delete activeUploads.value[key];
                    delete activeUploadProgress.value[key];

                    let response = {};
                    try {
                        response = xhr.responseText ? JSON.parse(xhr.responseText) : {};
                    } catch (error) {
                        response = {};
                    }

                    if (xhr.status >= 200 && xhr.status < 300) {
                        resolve({
                            path: response.Key ? response.Key.replace(`${bucket}/`, '') : path,
                            fullPath: response.Key || `${bucket}/${path}`,
                        });
                        return;
                    }

                    reject(
                        new Error(
                            response.message || response.error || `Supabase upload failed with status ${xhr.status}.`
                        )
                    );
                };

                xhr.onerror = () => {
                    const progress = formatUploadProgressForError(key);
                    delete activeUploads.value[key];
                    delete activeUploadProgress.value[key];
                    reject(new Error(`Supabase upload failed because of a network error. ${progress}`));
                };

                xhr.ontimeout = () => {
                    const progress = formatUploadProgressForError(key);
                    delete activeUploads.value[key];
                    delete activeUploadProgress.value[key];
                    reject(new Error(`Supabase upload timed out after ${timeoutSeconds} seconds. ${progress}`));
                };

                xhr.onabort = () => {
                    delete activeUploads.value[key];
                    delete activeUploadProgress.value[key];
                    reject(new Error('Supabase upload was cancelled.'));
                };

                xhr.send(file);
            });

        const abortActiveUploads = () => {
            Object.values(activeUploads.value || {}).forEach(xhr => xhr.abort());
            activeUploads.value = {};
            activeUploadProgress.value = {};
        };

        const formatUploadProgressForError = key => {
            const progress = activeUploadProgress.value?.[key];
            if (!progress) return '';
            return `Last progress: ${Math.round(progress.percentage)}% (${formatBytes(progress.loaded)} of ${formatBytes(
                progress.total
            )}).`;
        };

        const createSupabaseObjectUrl = (bucket, path) => {
            const supabaseUrl = String(props.content.supabaseUrl || '').trim().replace(/\/+$/, '');
            return `${supabaseUrl}/storage/v1/object/${encodeURIComponent(bucket)}/${encodeStoragePath(path)}`;
        };

        const createStoragePath = file => {
            const folder = cleanPath(props.content.supabasePath || '');
            const fileName = props.content.useOriginalFileName
                ? sanitizeFileName(file.name)
                : createUniqueFileName(file.name);
            return folder ? `${folder}/${fileName}` : fileName;
        };

        const createUniqueFileName = name => {
            const cleanName = sanitizeFileName(name);
            const dotIndex = cleanName.lastIndexOf('.');
            const baseName = dotIndex > 0 ? cleanName.slice(0, dotIndex) : cleanName;
            const extension = dotIndex > 0 ? cleanName.slice(dotIndex) : '';
            return `${baseName}-${Date.now()}-${Math.random().toString(36).slice(2, 8)}${extension}`;
        };

        const sanitizeFileName = name =>
            String(name || 'file')
                .trim()
                .replace(/[/\\?%*:|"<>]/g, '-')
                .replace(/\s+/g, '-')
                .replace(/-+/g, '-');

        const cleanPath = path =>
            String(path || '')
                .trim()
                .replace(/^\/+|\/+$/g, '')
                .split('/')
                .map(segment => sanitizeFileName(segment))
                .filter(Boolean)
                .join('/');

        const getPublicUrl = (bucket, path) => {
            const supabaseUrl = String(props.content.supabaseUrl || '').trim().replace(/\/+$/, '');
            return `${supabaseUrl}/storage/v1/object/public/${encodeURIComponent(bucket)}/${encodeStoragePath(path)}`;
        };

        const encodeStoragePath = path =>
            String(path)
                .split('/')
                .map(segment => encodeURIComponent(segment))
                .join('/');

        const toNonNegativeNumber = (value, fallback) => {
            const number = Number(value);
            return Number.isFinite(number) && number >= 0 ? number : fallback;
        };

        const formatBytes = bytes => {
            if (!Number.isFinite(bytes)) return '';
            if (bytes === 0) return '0 B';
            const units = ['B', 'KB', 'MB', 'GB'];
            const sizeIndex = Math.min(Math.floor(Math.log(bytes) / Math.log(1024)), units.length - 1);
            const value = bytes / 1024 ** sizeIndex;
            return `${value.toFixed(value >= 10 || sizeIndex === 0 ? 0 : 1)} ${units[sizeIndex]}`;
        };

        onBeforeUnmount(() => {
            abortActiveUploads();
        });

        const removeFile = index => {
            if (isDisabled.value || isReadonly.value) return;

            const updatedFiles = [...files.value.filter((_, i) => i !== index)];
            setFiles(updatedFiles);
            pruneStatus(updatedFiles);

            emitChange(updatedFiles);
        };

        const reorderFiles = (fromIndex, toIndex) => {
            if (isDisabled.value || isReadonly.value || !reorder.value) return;

            const newFiles = [...files.value];
            const [movedItem] = newFiles.splice(fromIndex, 1);
            newFiles.splice(toIndex, 0, movedItem);
            setFiles(newFiles);

            emitChange(newFiles);
        };

        const clearFiles = () => {
            abortActiveUploads();
            isUploading.value = false;
            setFiles([]);
            setStatus({});
            emitChange([]);
        };

        const getAllowedTypesLabel = () => {
            switch (extensions.value) {
                case 'image':
                    return 'Images';
                case 'video':
                    return 'Videos';
                case 'audio':
                    return 'Audio files';
                case 'pdf':
                    return 'PDF files';
                case 'csv':
                    return 'CSV files';
                case 'excel':
                    return 'Excel files';
                case 'word':
                    return 'Word documents';
                case 'json':
                    return 'JSON files';
                case 'custom':
                    return customExtensions.value;
                default:
                    return 'All files';
            }
        };

        wwLib.wwElement.useRegisterElementLocalContext('fileUpload', localData.value.fileUpload, {
            clearFiles: {
                description: 'Clear all files',
                method: clearFiles,
                editor: { label: 'Clear Files', group: 'File Upload', icon: 'trash' },
            },
            startUploading: {
                description: 'Upload pending files to Supabase',
                method: startUploading,
                editor: { label: 'Start Uploading', group: 'File Upload', icon: 'upload' },
            },
        });

        watch(
            isReadonly,
            value => {
                if (value) {
                    emit('add-state', 'readonly');
                } else {
                    emit('remove-state', 'readonly');
                }
            },
            { immediate: true }
        );

        watch(
            isDisabled,
            value => {
                if (value) {
                    emit('add-state', 'disabled');
                } else {
                    emit('remove-state', 'disabled');
                }
            },
            { immediate: true }
        );

        // Connect drag animation handlers without blocking WeWeb editor canvas drags.
        const handleDragOver = event => {
            if (isDisabled.value || isReadonly.value || !drop.value || isEditing.value) return;
            event.preventDefault();
            animationHandleDragOver(event);
        };
        const handleDragLeave = event => {
            if (isDisabled.value || isReadonly.value || !drop.value || isEditing.value) return;
            event.preventDefault();
            animationHandleDragLeave(event);
        };
        const handleMouseMove = animationHandleMouseMove;

        return {
            status,
            fileInput,
            isDragging,
            fileList,
            hasFiles,
            isDisabled,
            isReadonly,
            acceptedFileTypes,
            extensionsMessage,
            maxFileMessage,
            openFileExplorer,
            handleDragOver,
            handleDragLeave,
            handleDrop,
            handleFileSelection,
            removeFile,
            reorderFiles,
            getAllowedTypesLabel,
            iconHTML,
            uploadIconPosition,
            handleMouseMove,
            extensionsMessageStyle,
            maxFileMessageStyle,
            labelMessage,
            labelMessageStyle,
            type,
            reorder,
            drop,
            maxFileSize,
            minFileSize,
            maxFiles,
            required,
            extensions,
            customExtensions,
            showUploadIcon,
            uploadIcon,
            uploadIconColor,
            uploadIconSize,
            uploadIconMargin,
            enableCircleAnimation,
            circleSize,
            circleColor,
            circleOpacity,
            animationSpeed,

            safeDropzoneBorderWidth,
            safeDropzoneBorderStyle,
            safeDropzoneBorderColor,
            safeDropzoneBorderRadius,
            safeDropzonePadding,
            safeDropzoneMinHeight,
            safeDropzoneBackground,
            safeDropzoneBackgroundHover,
            safeDropzoneBackgroundDragging,
            safeLabelFontSize,
            safeLabelFontFamily,
            safeLabelFontWeight,
            safeLabelColor,
            safeFileDetailsFontSize,
            safeFileDetailsColor,
            safeProgressBarColor,
            safeLabelMarginBottom,
            safeProgressBarBackground,
            dropzoneEl,
            circleEl,
            mouseX,
            mouseY,
            targetX,
            targetY,
            isAnimating,
            isProcessing,
            isUploading,
            isEditing,

            clearFiles,
            startUploading,

            /* wwEditor:start */
            selectParentElement,
            /* wwEditor:end */
        };
    },
};
</script>

<style lang="scss" scoped>
.ww-file-upload {
    display: flex;
    flex-direction: column;
    width: 100%;
    position: relative;

    &__input {
        opacity: 0;
        background: rgba(0, 0, 0, 0);
        border: 0;
        bottom: -1px;
        font-size: 0;
        height: 1px;
        left: 0;
        outline: none;
        padding: 0;
        position: absolute;
        right: 0;
        width: 100%;
    }

    &__dropzone {
        position: relative;
        overflow: hidden;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        background-color: v-bind('safeDropzoneBackground');
        border: v-bind('safeDropzoneBorderWidth') v-bind('safeDropzoneBorderStyle') v-bind('safeDropzoneBorderColor');
        border-radius: v-bind('safeDropzoneBorderRadius');
        padding: v-bind('safeDropzonePadding');
        min-height: v-bind('safeDropzoneMinHeight');
        cursor: v-bind('isEditing ? "unset" : "pointer"');
        transition: all 0.3s ease;
        isolation: isolate;

        &:hover {
            border-color: v-bind('safeDropzoneBorderColor');
            background-color: v-bind('safeDropzoneBackgroundHover');
        }
    }

    &--has-files &__dropzone {
        margin-bottom: 16px;
    }

    &__content {
        display: flex;
        align-items: center;
        justify-content: center;
        text-align: center;
        width: 100%;
        pointer-events: none;
        position: relative;
        z-index: 2;

        &--top {
            flex-direction: column;
        }

        &--right {
            flex-direction: row-reverse;
            justify-content: center;
            text-align: right;
        }

        &--bottom {
            flex-direction: column-reverse;
        }

        &--left {
            flex-direction: row;
            justify-content: center;
            text-align: left;
        }
    }

    &__text {
        display: flex;
        flex-direction: column;
        pointer-events: none;
    }

    &__icon {
        font-size: v-bind('uploadIconSize');
        color: v-bind('uploadIconColor');
        width: 1em;
        height: 1em;
        display: flex;
        justify-content: center;
        align-items: center;
        flex-shrink: 0;
        pointer-events: none;
        margin: v-bind('uploadIconMargin');

        > :deep(svg) {
            width: 100%;
            height: 100%;
        }
    }

    &__info {
        display: block;
    }

    &--dragging {
        .ww-file-upload__dropzone {
            background-color: v-bind('safeDropzoneBackgroundDragging');
        }
    }

    &--disabled {
        opacity: 0.6;
        pointer-events: none;

        .ww-file-upload__dropzone {
            cursor: not-allowed;
            background-color: v-bind('safeDropzoneBackground');
        }
    }

    &--readonly {
        .ww-file-upload__dropzone {
            cursor: default;
            background-color: v-bind('safeDropzoneBackground');
        }
    }

    &__hover-circle {
        position: absolute;
        border-radius: 50%;
        pointer-events: none;
        z-index: -1;
        transform: translate(-50%, -50%);
        transition: opacity 0.3s ease-out, transform 0.3s ease-out;
        will-change: transform, opacity;
        box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
        backface-visibility: hidden;
    }
}
</style>
