<script setup lang="ts">
import { computed, ref } from 'vue'
import type { Comment, Attachment } from '@/stores/chat'
import { formatTimestamp } from '@/lib/utils'
import { FileText, Download, Play, Image as ImageIcon } from 'lucide-vue-next'
import {
    Dialog,
    DialogContent,
    DialogHeader,
    DialogTitle,
} from '@/components/ui/dialog'
import { Button } from '@/components/ui/button'

const props = defineProps<{
    message: Comment
}>()

const isAgentMessage = computed(() => {
    return props.message.sender.includes('agent')
})

const isCustomerMessage = computed(() => {
    return props.message.sender.includes('customer')
})

const isSystemMessage = computed(() => {
    return props.message.sender.includes('admin')
})

const showImagePreview = ref(false)
const selectedImage = ref<Attachment | null>(null)

const formatFileSize = (bytes: number) => {
    if (bytes === 0) return '0 Bytes'
    const k = 1024
    const sizes = ['Bytes', 'KB', 'MB', 'GB']
    const i = Math.floor(Math.log(bytes) / Math.log(k))
    return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}

const openImagePreview = (attachment: Attachment) => {
    selectedImage.value = attachment
    showImagePreview.value = true
    console.log('Opening image preview:', attachment)
}

const downloadFile = (attachment: Attachment) => {
    window.open(attachment.url, '_blank')
}

const renderAttachment = (attachment: Attachment) => {
    switch (attachment.type) {
        case 'image':
            return {
                component: 'image',
                icon: ImageIcon,
                action: () => openImagePreview(attachment),
                showThumbnail: true,
                buttonText: 'View'
            }
        case 'video':
            return {
                component: 'video',
                icon: Play,
                action: () => downloadFile(attachment),
                showThumbnail: true,
                buttonText: 'Play'
            }
        case 'pdf':
            return {
                component: 'file',
                icon: FileText,
                action: () => downloadFile(attachment),
                showThumbnail: false,
                buttonText: 'Download'
            }
        default:
            return {
                component: 'file',
                icon: FileText,
                action: () => downloadFile(attachment),
                showThumbnail: false,
                buttonText: 'Download'
            }
    }
}

const getAttachmentConfig = (attachment: Attachment) => renderAttachment(attachment)

const getImageUrl = (attachment: Attachment) => {
    return attachment.url || attachment.thumbnail_url || ''
}

const getThumbnailUrl = (attachment: Attachment) => {
    return attachment.thumbnail_url || attachment.url || ''
}
</script>

<template>
    <div class="flex w-full" :class="{
        'justify-start': isCustomerMessage,
        'justify-end': isAgentMessage,
        'justify-center': isSystemMessage
    }">
        <div class="rounded-2xl mb-1" :class="{
            'bg-[#f5f5f5] rounded-tl-md p-3 mr-10 lg:max-w-6/12': isCustomerMessage,
            'bg-[#21978b] rounded-tr-md p-3 ml-10 lg:max-w-6/12': isAgentMessage,
            'bg-[#f5f5f5] rounded-md p-2 mr-0 lg:max-w-5/12': isSystemMessage
        }">
            <!-- Message Text -->
            <p v-if="message.message" :class="{
                'text-[#141414] text-sm mb-2': isCustomerMessage && message.attachments.length > 0,
                'text-[#141414] text-sm': isCustomerMessage && message.attachments.length === 0,
                'text-white text-sm mb-2': isAgentMessage && message.attachments.length > 0,
                'text-white text-sm': isAgentMessage && message.attachments.length === 0,
                'text-[#4f5665] text-xs': isSystemMessage
            }">{{ message.message }}</p>

            <!-- Attachments -->
            <div v-if="message.attachments && message.attachments.length > 0" class="space-y-2">
                <div v-for="attachment in message.attachments" :key="attachment.id">
                    <template v-if="getAttachmentConfig(attachment).component === 'image'">
                        <!-- Image Attachments -->
                        <div class="relative group cursor-pointer" @click="getAttachmentConfig(attachment).action">
                            <img :src="getThumbnailUrl(attachment)" :alt="attachment.filename || 'Image'"
                                class="max-w-full max-h-64 rounded-lg hover:opacity-90 transition-opacity"
                                @error="$event.target.src = 'https://via.placeholder.com/300x200?text=Image+Not+Found'" />
                            <div class="mt-1 text-xs" :class="{
                                'text-[#4f5665]': isCustomerMessage,
                                'text-white/80': isAgentMessage
                            }">
                                {{ attachment.filename || 'Image' }} • {{ formatFileSize(attachment.size || 0) }}
                            </div>
                        </div>
                    </template>

                    <template v-else-if="getAttachmentConfig(attachment).component === 'video'">
                        <!-- Video Attachments -->
                        <div class="relative">
                            <div class="relative bg-gray-200 rounded-lg overflow-hidden cursor-pointer"
                                @click="getAttachmentConfig(attachment).action">
                                <img v-if="attachment.thumbnail_url" :src="attachment.thumbnail_url"
                                    :alt="attachment.filename || 'Video'" class="w-full h-40 object-cover"
                                    @error="$event.target.src = 'https://via.placeholder.com/320x180?text=Video+Thumbnail'" />
                                <div v-else class="w-full h-40 bg-gray-300 flex items-center justify-center">
                                    <Play class="w-12 h-12 text-gray-500" />
                                </div>

                            </div>
                            <div class="mt-2 flex items-center justify-between">
                                <div class="text-xs" :class="{
                                    'text-[#4f5665]': isCustomerMessage,
                                    'text-white/80': isAgentMessage
                                }">
                                    {{ attachment.filename || 'Video' }} • {{ formatFileSize(attachment.size || 0) }}
                                    <span v-if="attachment.duration"> • {{ attachment.duration }}s</span>
                                </div>
                                <Button size="sm" variant="ghost" class="h-6 px-2 text-xs" :class="{
                                    'text-[#21978b] hover:bg-gray-200': isCustomerMessage,
                                    'text-white hover:bg-white/20': isAgentMessage
                                }" @click="getAttachmentConfig(attachment).action">
                                    <component :is="getAttachmentConfig(attachment).icon" class="w-3 h-3 mr-1" />
                                    {{ getAttachmentConfig(attachment).buttonText }}
                                </Button>
                            </div>
                        </div>
                    </template>

                    <template v-else>
                        <!-- PDF and File Attachments -->
                        <div class="border rounded-lg p-3" :class="{
                            'border-gray-300 bg-white': isCustomerMessage,
                            'border-white/30 bg-white/10': isAgentMessage
                        }">
                            <div class="flex items-center space-x-3">
                                <div class="flex-shrink-0">
                                    <component :is="getAttachmentConfig(attachment).icon" class="w-8 h-8" :class="{
                                        'text-red-500': attachment.type === 'pdf',
                                        'text-[#21978b]': isCustomerMessage && attachment.type !== 'pdf',
                                        'text-white': isAgentMessage && attachment.type !== 'pdf'
                                    }" />
                                </div>
                                <div class="flex-1 min-w-0">
                                    <div class="text-sm font-medium truncate" :class="{
                                        'text-[#141414]': isCustomerMessage,
                                        'text-white': isAgentMessage
                                    }">
                                        {{ attachment.filename || 'File' }}
                                    </div>
                                    <div class="text-xs" :class="{
                                        'text-[#4f5665]': isCustomerMessage,
                                        'text-white/80': isAgentMessage
                                    }">
                                        {{ formatFileSize(attachment.size || 0) }}
                                        <span v-if="attachment.page_count"> • {{ attachment.page_count }} pages</span>
                                    </div>
                                </div>
                                <Button size="sm" variant="ghost" class="h-8 px-2" :class="{
                                    'text-[#21978b] hover:bg-gray-200': isCustomerMessage,
                                    'text-white hover:bg-white/20': isAgentMessage
                                }" @click="getAttachmentConfig(attachment).action">
                                    <Download class="w-4 h-4" />
                                </Button>
                            </div>
                        </div>
                    </template>
                </div>
            </div>

            <!-- Timestamp -->
            <div v-if="message.timestamp" class="text-xs mt-1 opacity-70" :class="{
                'text-[#4f5665]': isCustomerMessage,
                'text-white': isAgentMessage
            }">
                {{ formatTimestamp(message.timestamp) }}
            </div>
        </div>
    </div>

    <!-- Image Preview Dialog -->
    <Dialog v-model:open="showImagePreview">
        <DialogContent class="max-w-4xl max-h-[90vh] overflow-hidden p-0">
            <DialogHeader class="p-6 pb-2">
                <DialogTitle>{{ selectedImage?.filename || 'Image Preview' }}</DialogTitle>
            </DialogHeader>
            <div v-if="selectedImage" class="flex flex-col items-center space-y-4 p-6 pt-2">
                <div class="relative max-w-full max-h-[70vh] overflow-hidden rounded-lg bg-gray-100">
                    <img :src="getImageUrl(selectedImage)" :alt="selectedImage.filename || 'Image'"
                        class="max-w-full max-h-[70vh] object-contain rounded-lg"
                        @error="$event.target.src = 'https://via.placeholder.com/800x600?text=Image+Not+Available'"
                        @load="console.log('Image loaded successfully')" />
                </div>
                <div class="flex items-center space-x-4 text-sm text-gray-600">
                    <span>{{ formatFileSize(selectedImage.size || 0) }}</span>
                    <Button size="sm" @click="downloadFile(selectedImage)" class="bg-[#21978b] hover:bg-[#1a7a6f]">
                        <Download class="w-4 h-4 mr-2" />
                        Download
                    </Button>
                    <Button size="sm" variant="outline" @click="showImagePreview = false">
                        Close
                    </Button>
                </div>
            </div>
        </DialogContent>
    </Dialog>
</template>