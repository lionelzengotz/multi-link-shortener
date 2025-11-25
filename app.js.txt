class MultiLinkShortener {
    constructor() {
        this.historyKey = 'multiLinkHistory';
        this.init();
    }

    init() {
        this.bindEvents();
        this.loadHistory();
    }

    bindEvents() {
        document.getElementById('generateBtn').addEventListener('click', () => this.generateShortLink());
        document.getElementById('copyBtn').addEventListener('click', () => this.copyToClipboard());
        document.getElementById('clearHistoryBtn').addEventListener('click', () => this.clearHistory());
        
        // 支持Ctrl+Enter快速生成
        document.getElementById('linksInput').addEventListener('keydown', (e) => {
            if (e.key === 'Enter' && e.ctrlKey) {
                e.preventDefault();
                this.generateShortLink();
            }
        });
    }

    generateShortLink() {
        const linksInput = document.getElementById('linksInput').value.trim();
        const pageTitle = document.getElementById('pageTitle').value.trim() || '我的链接集合';
        const customSlug = document.getElementById('customSlug').value.trim();

        if (!linksInput) {
            this.showError('请输入至少一个链接');
            return;
        }

        // 解析多行链接
        const links = this.parseLinks(linksInput);
        
        if (links.length === 0) {
            this.showError('没有检测到有效的链接');
            return;
        }

        // 生成短链接和链接页面数据
        const shortLink = this.createShortLink(customSlug);
        const linkPageData = this.createLinkPageData(shortLink, pageTitle, links);
        
        this.displayResult(shortLink);
        this.saveToHistory(shortLink, pageTitle, links);
        this.saveLinkPageData(shortLink, linkPageData);
    }

    parseLinks(input) {
        const lines = input.split('\n');
        const validLinks = [];
        
        for (let line of lines) {
            const url = line.trim();
            if (url && this.validateUrl(url)) {
                validLinks.push({
                    url: url,
                    title: this.extractTitleFromUrl(url),
                    description: this.generateDescription(url)
                });
            }
        }
        
        return validLinks;
    }

    validateUrl(url) {
        try {
            new URL(url);
            return true;
        } catch {
            return false;
        }
    }

    extractTitleFromUrl(url) {
        try {
            const urlObj = new URL(url);
            const path = urlObj.pathname.split('/').pop() || '首页';
            return path.charAt(0).toUpperCase() + path.slice(1);
        } catch {
            return '未知页面';
        }
    }

    generateDescription(url) {
        if (url.includes('github.com')) return 'GitHub仓库页面';
        if (url.includes('docs')) return '文档页面';
        if (url.includes('contact')) return '联系我们页面';
        if (url.includes('download')) return '下载页面';
        return '相关页面';
    }

    createShortLink(customSlug) {
        const baseUrl = window.location.origin + '/link-page.html?id=';
        
        if (customSlug) {
            return baseUrl + this.sanitizeSlug(customSlug);
        } else {
            const randomSlug = Math.random().toString(36).substring(2, 10);
            return baseUrl + randomSlug;
        }
    }

    sanitizeSlug(slug) {
        return slug.toLowerCase().replace(/[^a-z0-9-_]/g, '');
    }

    createLinkPageData(shortLink, title, links) {
        return {
            id: shortLink,
            title: title,
            links: links,
            createdAt: new Date().toISOString()
        };
    }

    saveLinkPageData(shortLink, data) {
        const slug = shortLink.split('id=')[1] || shortLink;
        localStorage.setItem(`linkPage_${slug}`, JSON.stringify(data));
    }

    displayResult(shortLink) {
        const resultSection = document.getElementById('resultSection');
        const shortUrlInput = document.getElementById('shortUrlResult');
        const testLink = document.getElementById('testLink');

        shortUrlInput.value = shortLink;
        testLink.href = shortLink;
        
        resultSection.classList.remove('hidden');
        
        setTimeout(() => {
            resultSection.scrollIntoView({ behavior: 'smooth' });
        }, 100);
    }

    copyToClipboard() {
        const shortUrlInput = document.getElementById('shortUrlResult');
        
        shortUrlInput.select();
        shortUrlInput.setSelectionRange(0, 99999);
        
        try {
            navigator.clipboard.writeText(shortUrlInput.value).then(() => {
                this.showSuccess('短链接已复制到剪贴板！');
            });
        } catch (err) {
            document.execCommand('copy');
            this.showSuccess('短链接已复制到剪贴板！');
        }
    }

    saveToHistory(shortLink, title, links) {
        const history = this.getHistory();
        const historyItem = {
            id: Date.now(),
            shortLink: shortLink,
            title: title,
            linkCount: links.length,
            createdAt: new Date().toLocaleString()
        };

        history.unshift(historyItem);
        if (history.length > 50) history.pop();
        
        localStorage.setItem(this.historyKey, JSON.stringify(history));
        this.loadHistory();
    }

    getHistory() {
        const history = localStorage.getItem(this.historyKey);
        return history ? JSON.parse(history) : [];
    }

    loadHistory() {
        const history = this.getHistory();
        const historyList = document.getElementById('historyList');
        
        if (history.length === 0) {
            historyList.innerHTML = `
                <div class="text-center py-8 text-gray-500">
                    <i class="fas fa-inbox text-4xl mb-4"></i>
                    <p>暂无历史记录</p>
                </div>
            `;
            return;
        }

        historyList.innerHTML = history.map(item => `
            <div class="flex items-center justify-between p-4 bg-gray-50 rounded-lg border border-gray-200">
                <div class="flex-1">
                    <div class="flex items-center space-x-3 mb-2">
                        <span class="font-mono text-sm bg-blue-100 text-blue-800 px-2 py-1 rounded">
                            ${item.shortLink}
                        </span>
                        <span class="text-xs text-gray-500">
                            ${item.createdAt}
                        </span>
                    </div>
                    <p class="text-sm text-gray-600">
                        <i class="fas fa-folder mr-1 text-green-500"></i>
                        ${item.title} (${item.linkCount}个链接)
                    </p>
                </div>
                <div class="flex items-center space-x-2">
                    <button onclick="shortLinkGenerator.copyHistoryLink('${item.shortLink}')" 
                                class="text-blue-500 hover:text-blue-600 transition-colors duration-200">
                            <i class="fas fa-copy"></i>
                        </button>
                        <button onclick="shortLinkGenerator.deleteHistoryItem(${item.id})" 
                                class="text-red-500 hover:text-red-600 transition-colors duration-200">
                            <i class="fas fa-trash"></i>
                        </button>
                    </div>
                </div>
            `).join('');
    }

    copyHistoryLink(link) {
        navigator.clipboard.writeText(link).then(() => {
            this.showSuccess('短链接已复制！');
        });
    }

    deleteHistoryItem(id) {
        const history = this.getHistory().filter(item => item.id !== id);
        localStorage.setItem(this.historyKey, JSON.stringify(history));
        this.loadHistory();
    }

    clearHistory() {
        if (confirm('确定要清空所有历史记录吗？')) {
            localStorage.removeItem(this.historyKey);
            this.loadHistory();
            this.showSuccess('历史记录已清空');
        }
    }

    showError(message) {
        this.showNotification(message, 'error');
    }

    showSuccess(message) {
        this.showNotification(message, 'success');
    }

    showNotification(message, type = 'info') {
        const notification = document.createElement('div');
        const bgColor = type === 'error' ? 'bg-red-500' : 'bg-green-500';
        
        notification.className = `fixed top-4 right-4 ${bgColor} text-white px-6 py-3 rounded-lg shadow-lg transform transition-all duration-300 z-50`;
        notification.innerHTML = `
            <div class="flex items-center space-x-2">
                <i class="fas fa-${type === 'error' ? 'exclamation-triangle' : 'check-circle'}"></i>
                <span>${message}</span>
            </div>
        `;

        document.body.appendChild(notification);

        setTimeout(() => {
            notification.style.transform = 'translateX(100%)';
            setTimeout(() => {
                document.body.removeChild(notification);
            }, 300);
        }, 3000);
    }
}

const shortLinkGenerator = new MultiLinkShortener();