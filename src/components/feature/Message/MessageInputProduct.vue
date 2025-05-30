<script setup lang="ts">
import { ref } from 'vue'
import { storeToRefs } from 'pinia'
import { useChatStore, type Attachment } from '@/stores/chat'
import {
    Dialog,
    DialogContent,
    DialogDescription,
    DialogFooter,
    DialogHeader,
    DialogTitle,
    DialogTrigger,
} from '@/components/ui/dialog'
import { Button } from '@/components/ui/button'
import { FileText, Image as ImageIcon, Video, Upload } from 'lucide-vue-next'

const chatStore = useChatStore()
const { error } = storeToRefs(chatStore)

const isOpen = ref(false)
const fileInput = ref<HTMLInputElement | null>(null)

const sampleAttachments = ref([
    {
        id: 'sample_001',
        type: 'pdf' as const,
        filename: 'product_catalog.pdf',
        url: 'https://www.w3.org/WAI/ER/tests/xhtml/testfiles/resources/pdf/dummy.pdf',
        size: 1024000,
        mime_type: 'application/pdf',
        page_count: 10
    },
    {
        id: 'sample_002',
        type: 'image' as const,
        filename: 'product_image.jpg',
        url: 'https://picsum.photos/id/1/800/600',
        size: 245760,
        mime_type: 'image/jpeg',
        thumbnail_url: 'https://picsum.photos/id/1/200/150'
    },
    {
        id: 'sample_003',
        type: 'video' as const,
        filename: 'product_demo.mp4',
        url: 'https://sample-videos.com/zip/10/mp4/SampleVideo_720x480_1mb.mp4',
        size: 2048000,
        mime_type: 'video/mp4',
        duration: 45,
        thumbnail_url: 'https://picsum.photos/id/2/320/240'
    }
])

const sendAttachment = (attachment: Attachment) => {
    const success = chatStore.sendMessage(
        `Shared attachment: ${attachment.filename}`,
        undefined,
        [attachment]
    )

    if (success) {
        isOpen.value = false
    }
}

const handleFileUpload = (event: Event) => {
    const target = event.target as HTMLInputElement
    const files = target.files

    if (files && files.length > 0) {
        const file = files[0]

        const attachment: Attachment = {
            id: `upload_${Date.now()}`,
            type: getFileType(file.type),
            filename: file.name,
            url: URL.createObjectURL(file),
            size: file.size,
            mime_type: file.type
        }

        if (file.type.startsWith('image/')) {
            attachment.thumbnail_url = attachment.url
        }

        sendAttachment(attachment)
    }

    if (fileInput.value) {
        fileInput.value.value = ''
    }
}

const getFileType = (mimeType: string): 'image' | 'video' | 'pdf' | 'file' => {
    if (mimeType.startsWith('image/')) return 'image'
    if (mimeType.startsWith('video/')) return 'video'
    if (mimeType === 'application/pdf') return 'pdf'
    return 'file'
}

const getFileIcon = (type: string) => {
    switch (type) {
        case 'image': return ImageIcon
        case 'video': return Video
        case 'pdf': return FileText
        default: return FileText
    }
}

const formatFileSize = (bytes: number) => {
    if (bytes === 0) return '0 Bytes'
    const k = 1024
    const sizes = ['Bytes', 'KB', 'MB', 'GB']
    const i = Math.floor(Math.log(bytes) / Math.log(k))
    return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}

const triggerFileUpload = () => {
    fileInput.value?.click()
}
</script>

<template>
    <Dialog v-model:open="isOpen">
        <DialogTrigger as-child>
            <slot />
        </DialogTrigger>
        <DialogContent class="max-w-2xl max-h-[80vh] overflow-y-auto">
            <DialogHeader>
                <DialogTitle>Select Attachment</DialogTitle>
                <DialogDescription>
                    Choose from sample attachments or upload a new file to share with the customer.
                </DialogDescription>
            </DialogHeader>

            <div v-if="error" class="p-3 bg-red-100 border border-red-200 rounded-md mb-4">
                {{ error }}
            </div>

            <!-- Upload Section -->
            <div class="border-2 border-dashed border-gray-300 rounded-lg p-6 text-center mb-6">
                <Upload class="w-12 h-12 text-gray-400 mx-auto mb-4" />
                <h3 class="text-lg font-semibold text-gray-700 mb-2">Upload New File</h3>
                <p class="text-gray-500 mb-4">Upload images, videos, PDFs, or other files</p>
                <Button @click="triggerFileUpload" class="bg-[#21978b] hover:bg-[#1a7a6f]">
                    <Upload class="w-4 h-4 mr-2" />
                    Choose File
                </Button>
                <input ref="fileInput" type="file" class="hidden" accept="image/*,video/*,.pdf,.doc,.docx,.txt"
                    @change="handleFileUpload" />
            </div>

            <!-- Sample Attachments -->
            <div class="mb-4">
                <h3 class="text-lg font-semibold text-gray-700 mb-4">Sample Attachments</h3>
                <div class="space-y-3">
                    <div v-for="attachment in sampleAttachments" :key="attachment.id"
                        class="border border-gray-200 rounded-lg p-4 hover:shadow-md transition-shadow">
                        <div class="flex items-center space-x-4">
                            <div class="flex-shrink-0">
                                <component :is="getFileIcon(attachment.type)" class="w-10 h-10" :class="{
                                    'text-red-500': attachment.type === 'pdf',
                                    'text-blue-500': attachment.type === 'image',
                                    'text-purple-500': attachment.type === 'video',
                                    'text-gray-500': attachment.type === 'file'
                                }" />
                            </div>
                            <div class="flex-1 min-w-0">
                                <div class="text-sm font-medium text-gray-900 truncate">
                                    {{ attachment.filename }}
                                </div>
                                <div class="text-sm text-gray-500">
                                    {{ formatFileSize(attachment.size) }}
                                    <span v-if="attachment.page_count"> • {{ attachment.page_count }} pages</span>
                                    <span v-if="attachment.duration"> • {{ attachment.duration }}s</span>
                                </div>
                                <div class="text-xs text-gray-400 capitalize">
                                    {{ attachment.type }} file
                                </div>
                            </div>
                            <Button @click="sendAttachment(attachment)" size="sm"
                                class="bg-[#21978b] hover:bg-[#1a7a6f]">
                                Send
                            </Button>
                        </div>
                    </div>
                </div>
            </div>

            <div v-if="sampleAttachments.length === 0" class="text-center py-8 text-gray-500">
                <p>No sample attachments available.</p>
            </div>

            <DialogFooter>
                <Button variant="outline" @click="isOpen = false">
                    Cancel
                </Button>
            </DialogFooter>
        </DialogContent>
    </Dialog>
</template>