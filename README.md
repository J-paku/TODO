```
import React, { useState, useEffect } from 'react';
import { Modal } from '@mui/material';
import LoadingModal from "./LoadingModal";
import {hapticOn} from './hapticOn';
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import rehypeRaw from 'rehype-raw'; 
import type { InquiryEditorProps } from '../lib/useLoadCustomer';
import 'github-markdown-css/github-markdown.css';

const MarkdownViewer: React.FC<{ value: string }> = ({ value }) => {
  // [md]除去
  const cleanedValue = value.replace(/\[md\]/gi, '');

  // <br> 挿入（ステップタスクと同じフォーマット）
  const withLineBreaks = cleanedValue
    .split(/\r?\n/) // 改行で分割
    .map(line => line.trim()) // 各行の前後の空白を除去
    .join('<br>');
    

  // HTMLタグが含まれているか確認
  const isHtml = /<([a-z]+)([^>]*)>/i.test(withLineBreaks) && !withLineBreaks.includes('![');

  // const isHtml = /<\/?[a-z][\s\S]*>/i.test(withLineBreaks);

  return (
    <div className="">
      <div className="markdown-body text-left text-sm whitespace-pre-line">

        {/* imgタグがあればHTMLで表示、それ以外はMarkdown形式で表示 */}
        {isHtml ? (
          <ReactMarkdown
            remarkPlugins={[remarkGfm]}
            rehypePlugins={[rehypeRaw]}
          >
            {cleanedValue}
          </ReactMarkdown>
        ) : (
          <ReactMarkdown
            remarkPlugins={[remarkGfm]}
            rehypePlugins={[rehypeRaw]}
          >
            {cleanedValue}
          </ReactMarkdown>
        )}
      </div>
    </div>
  );
};


export default function InquiryEditor({ path, title, value, onChange }: InquiryEditorProps) {
  const [loading, setLoading] = useState(false);
  const convertLineBreaks = (html: string) => {
    return html
      .split(/\r?\n/)
      .map(line => line.trim())
      .join('<br>');
  };


  // ポップアップコンテンツを保存しているHook（原本）
  const [originalContent, setOriginalContent] = useState<string>('');

  // モーダルを開くときに元の内容を保存
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [tempContent, setTempContent] = useState(value);

  useEffect(() => {
    if(isModalOpen)
    {
      setOriginalContent(tempContent);
    }
  }, [isModalOpen]);

  const [selectedImage, setSelectedImage] = useState<string | null>(null); // 画像１枚ずつ
  const [selectedImages, setSelectedImages] = useState<string[]>([]); // 複数の画像

  // ポップアップ内、保存ボタンクリックイベント
  const handleSave = async () => {
    hapticOn('medium');

    // 画像ファイルが１枚を超えるとcombineImagesを実行する
    if(selectedImages.length > 0)
    {
      try {
        const combinedImage = await combineImages(selectedImages);

        // 既存イメージ除去
        const cleanedContent = tempContent.replace(/<img[^>]*data-clickable="true"[^>]*>/g, '');

        // 合体されたイメージ追加
        const imgTag = `<img src="${combinedImage}" style="width:100%;border-radius:8px;margin:4px;" data-clickable="true" />`;
        const newContent = cleanedContent + imgTag;
        

        setTempContent(newContent);
        onChange(newContent);
      } catch (error) {
        console.error('イメージ合体失敗:', error);
      }
    }
    else
    {
      onChange(tempContent);
      console.log(tempContent)
    }
    
      setIsModalOpen(false);
  };
  

  const handleImageUpload = (e: React.ChangeEvent<HTMLInputElement>) => {
    const files = e.target.files;

    if(files)
    {
      hapticOn('medium');
      Array.from(files).forEach((file) => {
        const reader = new FileReader();
        reader.onloadend = () => {
          const result = reader.result as string;
          if (result && result.startsWith('data:image')) {
            const imgTag = `<img src="${result}" data-clickable="true" />`;
            setTempContent((prev) => prev + imgTag);
            setSelectedImages((prev) => [...prev, result]);
          }
        };
        reader.readAsDataURL(file);
      });
    }
  };


  // 画像ファイルを合体するHook
  const [isComposing, setIsComposing] = useState(false);

  // 画像ファイルが複数の時、１枚にくっつけるロジック（ポップアップの保存ボタンで実行）
  const combineImages = async (images: string[]): Promise<string> => {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');

    const margin = 4;
    const maxPerRow = 1; // 分割数
    
    // モバイルFHD
    const maxWidth = 1080;
    const maxHeight = 1920;
    
    // イメージローディング＋ReSizing（画像ファイルが大きい際のみ、例え）iPhoneで撮った写真）
    const loadedImages = await Promise.all(images.map(src => {
      return new Promise<HTMLImageElement>((resolve, reject) => {
        const img = new Image();
        img.crossOrigin = 'anonymous'; // CORS防止
        img.onload = () => {
          if(img.naturalWidth > maxWidth || img.naturalHeight > maxHeight)
          {
            const ratio = Math.min(maxWidth / img.naturalWidth, maxHeight / img.naturalHeight);
            const resizedCanvas = document.createElement('canvas');
            resizedCanvas.width = img.naturalWidth * ratio;
            resizedCanvas.height = img.naturalHeight * ratio;
            const resizedCtx = resizedCanvas.getContext('2d');
            resizedCtx?.drawImage(img, 0, 0, resizedCanvas.width, resizedCanvas.height);

            const resizedImg = new Image();
            resizedImg.onload = () => resolve(resizedImg);
            resizedImg.onerror = reject;
            resizedCanvas.toBlob((blob: Blob | null) => {
            
              const resizedImg = new Image();
              resizedImg.onload = () => resolve(resizedImg);
              resizedImg.onerror = reject;
              resizedImg.src = URL.createObjectURL(blob!);
            }, 'image/jpeg', 0.95); // 解像度(0.1 ~ 1)
          } else {
            resolve(img);
          }
        };
        img.onerror = reject;
        img.src = src;
      });
    }));

    // 行を基準でサイズ調整する
    const rowHeights: number[] = [];
    const rowWidths: number[] = [];

    let currentRowWidth = 0;
    let currentRowMaxHeight = 0;

    loadedImages.forEach((img, index) => {
      currentRowWidth += img.width + (index % maxPerRow > 0 ? margin : 0);
      currentRowMaxHeight = Math.max(currentRowMaxHeight, img.height);

      if((index + 1) % maxPerRow === 0 || index === loadedImages.length - 1)
      {
        rowWidths.push(currentRowWidth);
        rowHeights.push(currentRowMaxHeight);
        currentRowWidth = 0;
        currentRowMaxHeight = 0;
      }
    });

    const canvasWidth = Math.max(...rowWidths);
    const canvasHeight = rowHeights.reduce((sum, h, i) => sum + h + (i > 0 ? margin : 0), 0);

    canvas.width = canvasWidth;
    canvas.height = canvasHeight;
    
    let y = 0;
    let imgIndex = 0;

    rowHeights.forEach((rowHeight) => {
      let x = 0;
      for (let i = 0; i < maxPerRow && imgIndex < loadedImages.length; i++, imgIndex++) {
        const img = loadedImages[imgIndex];
        ctx?.drawImage(img, x, y);
        x += img.width + margin;
      }
      y += rowHeight + margin;
    });

    return canvas.toDataURL('image/webp', 0.85); // 圧縮率85％（100％＝原本と一緒）
  };

  useEffect(() => {
    setTempContent(value);
  }, [value]);

  return (
    <div  className="flex items-center font-semibold border-b text-black border-gray-400">
      {typeof window !== "undefined" && loading && <LoadingModal />}
      <label className="w-24 mb-2">{title}</label>
      <div
        onClick={() => {
            setIsModalOpen(true);
            hapticOn();
        }}
        className={`relative w-full ${
          path === "Claim-History"
            ? "min-h-[30vh] max-h-[72vh]"
            : "min-h-[12vh] max-h-[12vh]"
        } overflow-y-auto`}
      >
        {path === 'Claim-History' && (
          <div>
            {value.trim() === '' ? (
              <div style={{ color: '#999' }}>読み込み中...</div>
            ) : (
              <MarkdownViewer
                value={value}
              />
            )}
          </div>
        )}

        {/* テキストエリアHTMLタグ除去 */}
        <>
          {path !== 'Claim-History' && (
            <div
              className="p-2 pb-4 text-black"
              dangerouslySetInnerHTML={{ __html: value }}
            />
          )}
        </>

      </div>
      <Modal
        open={isModalOpen}
        onClose={() => {
          setIsModalOpen(false);
          setTempContent(originalContent);
        }}
      >
        
        <div
          className="fixed top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2
                     flex flex-col w-[95vw] h-[85vh] bg-gray-300 border-2 border-gray-600 p-3 rounded-lg"
        >
          <div className="flex-grow mb-2"> 
            <div className="flex justify-end">
              <button
                className="transition-colors"
                onClick={() => {
                  setTempContent(originalContent);
                  setIsModalOpen(false);
                  hapticOn();
                }}
              >
                <span className="material-symbols-outlined">close</span>
              </button>
            </div>

            <div className='font-bold text-lg break-words text-center mb-2 -mt-8'>
              {title}
            </div>
            <div
              contentEditable
              className="w-full min-h-[72vh] max-h-[72vh] bg-white/20 rounded p-3 overflow-y-auto focus:outline-none
                        ring-2 ring-gray-400 focus:ring-gray-600 text-[16px]"
              dangerouslySetInnerHTML={{ __html: convertLineBreaks(tempContent) }}
              onBlur={(e) => {
                const html = (e.currentTarget as HTMLDivElement).innerHTML;
                setTempContent(html);
                // 現在のHTMLから画像srcを抽出して selectedImages を更新
                const parser = new DOMParser();
                const doc = parser.parseFromString(html, 'text/html');
                const imgElements = Array.from(doc.querySelectorAll('img[data-clickable="true"]'));
                const currentImageSrcs = imgElements.map(img => img.getAttribute('src')).filter(Boolean) as string[];
                setSelectedImages(currentImageSrcs);
              }}
              onCompositionEnd={() => {
                setIsComposing(false);
              }}
              onClick={(e) => {
                hapticOn();
                setIsModalOpen(true); 
                const target = e.target as HTMLElement;
                if(target.tagName === 'IMG' && target.dataset.clickable === 'true')
                {
                  setSelectedImage(target.getAttribute('src'));
                  // 画像をクリックするとキーボードレイアウト表示しない
                  const editableDiv = e.currentTarget as HTMLDivElement;
                  editableDiv.blur();
                }
              }}
            />
          </div>
            
          <div className="flex justify-between items-center px-4 mb-3">
            <div className="flex text-white rounded hover:bg-white/10 cursor-pointer">
              <button 
                className="flex items-center -mt-1"
                type="button"
                onClick={() => {
                  hapticOn();
                  document.getElementById("imageUploadInput")?.click();
                }}
              >
                <span className="material-symbols-outlined text-black" style={{ fontSize: '2rem', fontWeight: 'bold' }}>
                  wallpaper
                </span>
              </button>
              <input
                className="hidden"
                id="imageUploadInput"
                type="file"
                accept="image/*"
                onChange={handleImageUpload}
                multiple
              />
            </div>
              
            <div className="flex justify-center flex-1 mt-1">
              <button
                className="flex items-center text-white px-4 py-2 mr-3 rounded-sm bg-blue-500 hover:bg-blue-600 transition-all duration-200 shadow-md"
                onClick={handleSave}
              >
                <span className="material-symbols-outlined mr-2">save</span>
                保存
              </button>
            </div>
          </div>
          {/* プレビュー拡大 */}
          {selectedImage && (
            <div
              className="fixed top-0 left-0 w-full h-full bg-black/70 flex items-center justify-center z-50 cursor-pointer"
              onClick={() => setSelectedImage(null)}
            >
              <img
                src={selectedImage}
                alt="Expanded"
                className="max-w-[90vw] max-h-[90vh] rounded-lg shadow-lg"
              />
            </div>
          )}
        </div>
      </Modal>
    </div>
  );
}
```
```
 ```
  qweqwe
  qwqweqw
  qweqwe
 ```
할 때, 출력물에 <br>이 껴져있는 문제
```
