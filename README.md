```
import React, { useEffect, useState, useMemo, useRef} from 'react';
import Header from '../components/Header';
import Head from 'next/head';
import DatePicker from "../components/DatePicker";
import ClientSearchModal from '../components/ClientSearchModal';
import ClientSelectorBox from '../components/ClientSelectorBox'
import ToastPopup from '../components/ToastPopup';
import {hapticOn} from '../components/hapticOn';
import LoadingModal from "../components/LoadingModal";
import { refreshCustomerData } from "../lib/useRefreshCustomer";
import { loadCustomerDataFromSessionStorage } from "../lib/useLoadCustomer";
import type { Client, Customer } from '../lib/useLoadCustomer';
import {steptaskGetTouenHistory, steptaskDeleteTouenHistory} from '../lib/fetchSteptask';
import { GetStaticProps } from "next";
import useSWR from 'swr';
import HistoryPopUpModal from '../components/HistoryPopUpModal';
import { haversine } from '../lib/haversine';
import { useInitializeUser } from '../hook/useInitializeUser';
import {hankakuToZenkakuKatakana} from '../lib/convenientFunction';

const fetchTouenData = async (): Promise<Client[]> => {
  const data = await steptaskGetTouenHistory() as any[];

  return data.map((item: any, index: number) => {
    const createdTime = item?.DateHash?.DateA ?? "";
    const dateObj = new Date(createdTime);

    const formattedDate = dateObj.toLocaleDateString("ja-JP", {
      year: "numeric",
      month: "2-digit",
      day: "2-digit",
    });

    const formattedTime = dateObj.toLocaleTimeString("ja-JP", {
      hour: "2-digit",
      minute: "2-digit",
      hour12: false,
    });

    return {
      idx: index + 1,
      item: item?.DescriptionHash?.DescriptionE,
      qty: item?.NumHash?.NumA,
      price: item?.NumHash?.NumB,
      clients_name: item?.DescriptionHash?.DescriptionB ?? "不明な園",
      created_date: formattedDate,
      created_time: formattedTime,
      creator: item?.ClassHash?.ClassZ ?? "不明な担当者",
      ResultId: item?.ResultId,
      Identifier: item?.ClassHash?.ClassY,
    };
  });
};

export const getStaticProps: GetStaticProps = async () => {
  try {
    const data: any = await steptaskGetTouenHistory();
    alert(data);
    console.log("getStaticProps API 응답:", JSON.stringify(data));

    if(!Array.isArray(data))
    {
      console.error("配列ではありません:", data);
      return { props: { initialClients: [] } };
    }

    const mappedClients = data.map((item: any, index: number) => {
      const createdTime = item?.DateHash?.DateA ?? "";
      const dateObj = new Date(createdTime);

      const formattedDate = dateObj.toLocaleDateString("ja-JP", {
        year: "numeric",
        month: "2-digit",
        day: "2-digit",
      });

      const formattedTime = dateObj.toLocaleTimeString("ja-JP", {
        hour: "2-digit",
        minute: "2-digit",
        hour12: false,
      });

      return {
        idx: index + 1,
        item: item?.DescriptionHash?.DescriptionE,
        qty: item?.NumHash?.NumA,
        price: item?.NumHash?.NumB,
        clients_name: item?.DescriptionHash?.DescriptionB ?? "不明な園",
        created_date: formattedDate,
        created_time: formattedTime,
        creator: item?.ClassHash?.ClassZ ?? "不明な担当者",
        ResultId: item?.ResultId,
        Identifier: item?.ClassHash?.ClassY,
      };
    });

    return {
      props: { initialClients: mappedClients },
    };
  } catch (error) {
    console.error("履歴取得エラー:", error);
    return { props: { initialClients: [] } };
  }
};


// 今日の日付
const getFirstDayOfMonth = () => {
  const today = new Date(
   new Date().toLocaleString("en-US", { timeZone: "Asia/Tokyo" })
  );
  return today.toLocaleDateString('ja-JP', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
  }).replaceAll('/', '-');
};

const getLastDayOfMonth = () => {
  const today = new Date(
    new Date().toLocaleString("en-US", { timeZone: "Asia/Tokyo" })
  );
  return today.toLocaleDateString('ja-JP', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
  }).replaceAll('/', '-');
};
// 今日の日付 END

const OmutuList: React.FC<{ initialClients: any[] }> = ({ initialClients }) => {
  useEffect(() => {
    const fetchData = async () => {
      try {
        const updatedClients = await fetchTouenData();
        setAllClients(updatedClients);
        console.log("クライアントで再取得したデータ:", updatedClients);
      } catch (error) {
        console.error("クライアント側データ取得エラー:", error);
      }
    };

    fetchData();
  }, []);

  const [allClients, setAllClients] = useState<any[]>(initialClients); 
  const [loading, setLoading] = useState(false);

  // ClientSearchModal.tsxに更新時に必要な情報を渡す
  const getGpsAndSteptaskAPI = () => {
    setCustomers([]);  // リスト初期値
    setNearestClientName("全件");  // 全件がdefault
  
    refreshCustomerData(
      setCustomers,
      setUserLatitude,
      setUserLongitude,
      setPrintStatus,
      setShowToast,
      setLoading
    );
  };
  // ClientSearchModal.tsxに更新時に必要な情報を渡す END
  
  // 現在位置GPS取得
  const [modalVisible, setModalVisible] = useState(false);
  const [searchText, setSearchText] = useState('');
  const [customers, setCustomers] = useState<Customer[]>([]);
  
  const [userLatitude, setUserLatitude] = useState<number | null>(null);
  const [userLongitude, setUserLongitude] = useState<number | null>(null);
  // 現在位置GPS取得　END
  
  // Select-Option デフォルト値
  const [nearestClientName, setNearestClientName] = useState<string>("全件");
  const [moyoriSaki, setMoyoriSaki] = useState<string | null>(null);
  
  // Omutu-Count開くとき実行する関数
  useEffect(() => {
    const {
      customers,
      userLatitude,
      userLongitude,
      moyoriSaki
    } = loadCustomerDataFromSessionStorage();
  
    setCustomers(customers);
    setNearestClientName("全件");
    setMoyoriSaki(moyoriSaki);
    setUserLatitude(userLatitude);
    setUserLongitude(userLongitude);
  
    document.body.style.overflow = 'hidden';
    return () => {
      document.body.style.overflow = 'auto';
    };
  }, []);

  const sortedItems = useMemo(() => {
    return customers
      .map((client) => {
        if (userLatitude === null || userLongitude === null) return null;

        const distance = haversine(
          userLatitude,
          userLongitude,
          parseFloat(client.緯度),
          parseFloat(client.経度)
        );

        return {
          ...client,
          distance: (Math.ceil(distance * 100) / 100).toFixed(2),
        };
      })
      .filter((client): client is { distance: string } & typeof client => client !== null) // ✅ null除去 + タイプ指定
      .sort((a, b) => parseFloat(a.distance) - parseFloat(b.distance));
  }, [customers, userLatitude, userLongitude]);

  // Omutu-Count開くとき実行する関数　END

  // カレンダー関連
  const [startDate, setStartDate] = useState<Date>(new Date());
  const [endDate, setEndDate] = useState<Date>(new Date());
  // Toast Popup
  const [showToast, setShowToast] = useState(false); 
  const [printStatus, setPrintStatus] = useState<string | null>(""); // トーストポップアップテキスト
  
  useEffect(() => {
    const firstDay = getFirstDayOfMonth();
    const lastDay = getLastDayOfMonth();
    
    setStartDate(new Date(firstDay));
    setEndDate(new Date(lastDay));
  }, []);
  // Toast Popup END

  const stripTime = (date: Date) => {
    return new Date(date.getFullYear(), date.getMonth(), date.getDate());
  };

  const handleStartDateChange = (date: Date) => {
    const newStartDate = stripTime(date);

    setPrintStatus(process.env.NEXT_PUBLIC_SUCCESS_MESSAGE as string);
    setStartDate(newStartDate);
    setShowToast(true);
    hapticOn(); // スマホに振動機能haptic

  };

  const handleEndDateChange = (date: Date) => {
    const newEndDate = stripTime(date);

    setEndDate(newEndDate);
    setPrintStatus(process.env.NEXT_PUBLIC_SUCCESS_MESSAGE as string);
    setShowToast(true);
    hapticOn(); // スマホに振動機能haptic
  };

  // カレンダー関連 END

  // モダールが開くとスクロールできる現状除去
  useEffect(() => {
    document.body.style.overflow = 'hidden';
    document.documentElement.style.overflow = 'hidden';

    return () => {
      document.body.style.overflow = '';
      document.documentElement.style.overflow = '';
    };
  }, [modalVisible]);
  // モダールが開くとスクロールできる現状除去END
  // 表 Start
    // データ表示（ユーザー名で判定）
    const { userInfo } = useInitializeUser(); // StepTaskユーザー情報取得
    const sessionUserName = userInfo?.Name;
    const [hasLogged, setHasLogged] = useState(false);
    
    useEffect(() => {
      if (userInfo && !hasLogged) {
        // console.log(userInfo, sessionUserName);
        setHasLogged(true);
      }
    }, [userInfo, hasLogged]);

    const [showOnlyMine, setShowOnlyMine] = useState<boolean>(true);


  const { data: clients = [], mutate } = useSWR('touen-history', fetchTouenData,
  {
    fallbackData: initialClients, // 初期表示用のデータ。APIから取得できない場合に使用される。
    refreshInterval: 5000,       // 5秒ごとに自動的にデータを再取得する。
    refreshWhenHidden: false,     // タブが非表示のときは自動更新を停止する。
    // revalidateOnFocus: false,     // ユーザーが画面に戻ったときに再取得しないようにする。
  })
  mutate();
// filteredClients
  // 表のデータをフィルタする
  const filteredClients = useMemo(() => {
    const grouped: { [key: string]: any[] } = {};
    
    clients.forEach((client) => {
      const identifier = client.Identifier;
      if (!grouped[identifier]) {
        grouped[identifier] = [];
      }
      grouped[identifier].push(client);
    });
  
    // グループごとにフィルターを適用
    return Object.values(grouped).map((group) => {
      return group.filter((client) => {
        const matchUser = showOnlyMine ? client.creator === sessionUserName : true;
      
        const clientDate = new Date(client.created_date.replace(/\//g, '-'));
        const endOfDay = new Date(endDate);
        endOfDay.setHours(23, 59, 59, 999);
      
        const inRange = clientDate >= startDate && clientDate <= endOfDay;
        const matchClientName = nearestClientName === "全件" || client.clients_name === nearestClientName;
      
        return matchUser && inRange && matchClientName;
      });
    }).filter(group => group.length > 0); // 空グループは除外
  }, [clients, startDate, endDate, showOnlyMine, sessionUserName, nearestClientName]);
  // 表のデータをフィルタする END


  // データ表示（ユーザー名で判定）END

  //  履歴削除モダール
    // ユーザーが画面をスクロールしたかどうかを判定するために、タッチ開始時のY座標を保持する
    const touchStartY = useRef<number | null>(null);
    // 確認モーダルを表示するかどうかの状態管理
    const [showConfirmModal, setShowConfirmModal] = useState(false);
    // ユーザーが選択したクライアント情報を保持する状態
    const [selectedClient, setSelectedClient] = useState<Client[]>([]);
    // 現在押されている行のインデックスを保持し、視覚的なフィードバック（背景色など）を提供する
    const [pressingIndex, setPressingIndex] = useState<number | null>(null);
    // 長押し判定のためのタイマーを保持する。setTimeoutの戻り値を格納
    const pressTimer = useRef<number | null>(null);
    const pressStartTime = useRef<number | null>(null);
    const isLongPressTriggered = useRef(false);
    // ユーザーがスクロールしているかどうかを判定するためのフラグ
    const isScrolling = useRef<boolean>(false);

    const [tooltip, setTooltip] = useState<boolean>(true);

    const graphStyleReset = () => {
        setTooltip(true);
        setPressingIndex(null);
    }

    // ユーザーが行を押し始めたときに呼ばれる関数（マウスまたはタッチ）
    const PressShowModalStart = (index: number, e: React.TouchEvent | React.MouseEvent) => {
      hapticOn('light');
      const client = filteredClients[index]; // ⭐️⭐️⭐️⭐️⭐️　オブジェクトのIndexを原本データから使う
      // console.log(client);

      setTooltip(false);
      isScrolling.current = false;
      isLongPressTriggered.current = false;
      setPressingIndex(index);
      pressStartTime.current = Date.now();

      const target = e.currentTarget as HTMLElement;

      const touchMoveHandler = () => {
        isScrolling.current = true;
        setPressingIndex(null);
        if(pressTimer.current)
        {
          clearTimeout(pressTimer.current);
          pressTimer.current = null;
        }
      }
      // console.log(client)
    
      if('touches' in e)
      {
        target.addEventListener('touchmove', touchMoveHandler, { passive: true });
      }
    
      pressTimer.current = window.setTimeout(() => {
        if(!isScrolling.current)
        {
          hapticOn('medium');
          isLongPressTriggered.current = true;
          setSelectedClient(client as any);
          
          setShowConfirmModal(true);
        }
      
        if('touches' in e)
        {
          target.removeEventListener('touchmove', touchMoveHandler);
        }
      }, 350);
    };

    // ユーザーが指を離したとき（またはマウスボタンを離したとき）に呼ばれる関数
    const PressShowModalEnd = (e: React.TouchEvent | React.MouseEvent) => {
      setTooltip(false);
        
      if(pressTimer.current)
      {
        clearTimeout(pressTimer.current);
        pressTimer.current = null;
      }
    
      const duration = Date.now() - (pressStartTime.current ?? 0);
    
      // 長押しが３５０ms以下行われたら何もしない
      if(!isLongPressTriggered.current && duration < 350)
      {
        setPressingIndex(null);
        return;
      }
    
      setPressingIndex(null);
    };
    // 表 End

    // ToolTipが４秒表示されたら消す
    useEffect(() => {
      if (!tooltip) {
        setTimeout(()=> {
          setTooltip(true);
        },4000)
      }
    }, [tooltip]);
    // ToolTipが４秒表示されたら消す　END

    // レコード削除ロジック
    const deleteRecord = async (selectedClients: Pick<Client, 'idx' | 'clients_name' | 'created_date' | 'created_time' | 'creator' | 'ResultId'>[]) => {
      setShowConfirmModal(false);

      for(const client of selectedClients) {
        const isMyRecord = selectedClient[0].creator === sessionUserName;

        if(!isMyRecord)
        {
          hapticOn("medium");
          setShowToast(true);
          setPrintStatus(`${process.env.NEXT_PUBLIC_WARNING_MESSAGE}（他人が登録したデータの為）`);
        }
        else
        {
          setLoading(true);
          await steptaskDeleteTouenHistory(client.ResultId);
          mutate(); // SWRで最新データ持ってくる
          setLoading(false);
          hapticOn("success");
          setShowToast(true);
          setPrintStatus(`${process.env.NEXT_PUBLIC_PRINT_DELETE_MESSAGE}`);
        }
      }
    };
    // レコード削除ロジック END
  //  履歴削除モダール End

  const [second, setSecond] = useState(3000);
  const centerAlign = (text: string, width: number): string => {
    const padding = Math.floor((width - text.length) / 2);
    return ' '.repeat(Math.max(padding, 0)) + text;
  };

  const reprintEventBtn = ( clientName: string, updatedTime: string, historyItems: { item: string; qty: number;}[],) => {
    console.log("重要！", allClients);
    console.log(historyItems);
    const kaiGyou = "\n";
    const kaiGyouTwo = " \n \n \n ";

    const title = "納品書 (KMS)" + " \n ";
    const dateTime = (centerAlign(updatedTime, 45) + " \n ").replace("T"," ");
    const clientLine = `${clientName} 様`;
    const productLine = "--------------------------------";

    // const itemLines = historyItems.length === 0 ? [``] :
    // historyItems.map(item => {
    //   const name = hankakuToZenkakuKatakana(item?.item || "").padEnd(14, '　');
    //   const qty = String(item.qty);
    //   return `${name} :  ${qty}  ${kaiGyou} ${kaiGyou}`;
    // });
    try {
      const itemLines = historyItems.length === 0 ? [``] :
        historyItems.map(item => {
          const name = hankakuToZenkakuKatakana(item?.item || "").padEnd(14, '　');
          const qty = String(item.qty);
          return `${name} :  ${qty}  ${kaiGyou} ${kaiGyou}`;
        });
      console.log("itemLines:", itemLines);
      console.log("itemLines raw:", JSON.stringify(itemLines));

      const printDescription = [
        title,
        kaiGyou,
        dateTime,
        clientLine,
        kaiGyou,
        productLine,
        ...itemLines,
        productLine,
        kaiGyouTwo,
      ].join("\n");
      console.log("printDescription1:", (printDescription));
      console.log("printDescription2:", JSON.stringify(printDescription));

      const win = window as any;

      let retryInterval: NodeJS.Timeout;
      let startTime = Date.now();
      let retryCount = 0;
      const maxRetries = 18 + 1;

      const tryPrint = () => {
        retryCount++;
        const remaining = maxRetries - retryCount;

        if(Date.now() - startTime > 90_000)
        {
          hapticOn("error");
          clearInterval(retryInterval);
          setSecond(3000);
          setPrintStatus("🖨️を見つかりませんでした。");
          setLoading(false);
          setShowToast(true);
          return;
        }

        if(win.webkit?.messageHandlers?.printHandler)
        {
          win.webkit.messageHandlers.printHandler.postMessage(printDescription);
          hapticOn("medium");
          setSecond(4700);
          setPrintStatus(`${process.env.NEXT_PUBLIC_PRINT_WAIT_MESSAGE} ${remaining}回`);
        }
        else
        {
          hapticOn("error");
          clearInterval(retryInterval);
          setSecond(3000);
          setPrintStatus("印刷失敗。iOS端末を確認してください。");
          setLoading(false);
          setShowToast(true);
        }
      };

      setLoading(true);

      tryPrint();
      retryInterval = setInterval(tryPrint, 5000);

      window.onPrintResult = function (message: string) {
        if(message.includes("✅"))
        {
          hapticOn("success");
          clearInterval(retryInterval);
          setPrintStatus("🖨️ 再印刷成功しました");
          setLoading(false);
          setShowToast(true);
        }
        else
        {
          const remaining = maxRetries - retryCount;
          setSecond(4800);
          setPrintStatus(`${process.env.NEXT_PUBLIC_PRINT_WAIT_MESSAGE} ${remaining}回`);
          setShowToast(true);
        }
      };
    } catch (e) {
      console.error("エラー:", e);
    }
  };

  return (
    <div>
      <Head><link rel="icon" href="../favicon.ico" /></Head>
      <Header onResetClientName={() => setNearestClientName("全件")} onResetJibun={()=> setShowOnlyMine(true)}/>
      {typeof window !== "undefined" && loading && <LoadingModal />}

      {showToast && (
        <ToastPopup
          message={printStatus ?? "メッセージがありません"}
          setToast={setShowToast}
          position={
            typeof window !== "undefined" && window.innerWidth >= 768
              ? "top"
              : "center"
          }
        />
      )}
      {/* 履歴削除モダール */}
      <HistoryPopUpModal
        show={showConfirmModal}
        client={selectedClient}
        loading={loading}
        onCancel={() => {
          setShowConfirmModal(false);
          hapticOn();
          graphStyleReset();
        }}
        onDelete={() => {
          if (!selectedClient) return;
                    
          setLoading(true);
          setTimeout(() => {
            hapticOn();
            deleteRecord(selectedClient);
            graphStyleReset();
            setLoading(false);
          }, 250);
        }}
        onRePrint={() => {
          if (!selectedClient || selectedClient.length === 0) return;

          const isIOS = typeof window !== 'undefined' && /iPad|iPhone|iPod/.test(navigator.userAgent);

          // 共通の時間情報抽出
          const updatedTime =
            selectedClient[0].created_date + 'T' + selectedClient[0].created_time;

          // 同じ時間＋同じ名前の項目フィルター
          const groupedItems = allClients.filter(
            item =>
              item.created_date + 'T' + item.created_time === updatedTime &&
              item.clients_name === selectedClient[0].clients_name
          );
          // if(!isIOS)
          // {
          //   hapticOn('error');
          //   const NEXT_PUBLIC_PRINT_FAIL_MESSAGE =
          //     process.env.NEXT_PUBLIC_PRINT_FAIL_MESSAGE || '';
          //   const PRINT_FAIL_MESSAGE = NEXT_PUBLIC_PRINT_FAIL_MESSAGE.split('　')[0];
          //   setPrintStatus(PRINT_FAIL_MESSAGE);
          //   setSecond(3000);
          //   setShowToast(true);
          //   return;
          // }
        
          // iOSのみプリント
          setLoading(true);
          setTimeout(() => {
            hapticOn();
            reprintEventBtn(selectedClient[0].clients_name, updatedTime, groupedItems);
            setLoading(false);
          }, 850);
        }}
      />

      {/* 得意先検索モダール */}
      {modalVisible && (
        <ClientSearchModal
          searchText={searchText}
          setSearchText={setSearchText}
          sortedItems={sortedItems}
          setNearestClientName={setNearestClientName}
          setModalVisible={setModalVisible}
          onRefresh={getGpsAndSteptaskAPI}
        />
      )}
      <div className="min-h-[100vh] bg-gray-100">
        <div className='p-4'>
          {/* 最寄り先名選択エリア */}
          <ClientSelectorBox
            nearestClientName={nearestClientName}
            nearestClientData={moyoriSaki}
            onOpenModal={() => setModalVisible(true)}
            onClientNameChange={(newName: string) => setNearestClientName(newName)}
          />
          {/* カレンダー選択エリア */}
          <div className="flex justify-around items-center mt-2 text-sm text-[16px]">
            <DatePicker
              label=""
              date={startDate}
              onChange={handleStartDateChange}
              maxDate={endDate}
            />
            <span className="mb-2 -mx-4">～</span>
            <DatePicker
              label=""
              date={endDate}
              onChange={handleEndDateChange}
              minDate={startDate}
            />
          </div>

          {/* チェックボックス */}
          <div className="flex items-center -mb-3 cursor-pointer select-none relative">
            <div 
              onClick={() => {
                setShowOnlyMine((prev) => !prev)
                hapticOn('medium');
              }}
            >
              <input
                type="checkbox"
                className="mr-1 pointer-events-none"
                checked={showOnlyMine}
                readOnly
              />
              自分
            </div>

            <div
              className={`absolute top-5 right-1 ml-5 -mt-1 p-1 border text-xs bg-black text-white ${
                  tooltip ? 'opacity-0 hidden' : 'opacity-80 font-bold'
              }`}
            >
              <p>
                {/* 💡得意先を長押しすると削除できます。 */}
                {process.env.NEXT_PUBLIC_OMUTU_LIST_EXPLAIN} 
              </p>
            </div>
          </div>
          
          {/* グリッドエリア */}
          <div className="bg-white mt-4 p-1 pr-2 rounded-lg shadow-md select-none overflow-y-scroll max-h-[60vh]">
            <div className="overflow-hidden rounded-lg border border-gray-700">
              <table className="min-w-full border-collapse">
                <thead>
                  <tr className="bg-[#558ED5] text-white font-bold">
                    <th className="border border-gray-600 w-[60%]">得意先名</th>
                    <th className="border border-gray-600 w-[25%]">日付</th>
                    <th className="border border-gray-600 w-[15%]">時間</th>
                  </tr>
                </thead>
                <tbody>
                  {filteredClients.map((client, index) => (
                    <tr
                      key={index}
                      className={`cursor-pointer text-sm select-none ${
                        pressingIndex === index
                          ? 'bg-gray-100'
                          : client[0].creator === sessionUserName
                          ? 'bg-[#FFFFD5]' // PiPiT画面案.pptxの指定色
                          : 'hover:bg-gray-100'
                      }`}
                      // onClick={() => handleTouch()}
                      onMouseDown={(e) => PressShowModalStart(index, e)}
                      onMouseUp={PressShowModalEnd}
                      onMouseLeave={PressShowModalEnd}
                      onTouchStart={(e) => {
                        touchStartY.current = e.touches[0].clientY;
                        PressShowModalStart(index, e);
                      }}
                      onTouchEnd={PressShowModalEnd}
                    >
                      <td className="p-2 text-center border border-black truncate max-w-[150px] sm:max-w-none" title={client[0].clients_name}>
                        {client[0].clients_name}
                      </td>
                      <td className="text-center border border-gray-600 text-xs">{client[0].created_date}</td>
                      <td className="text-center border border-gray-600 text-xs">{client[0].created_time}</td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

export default OmutuList;

위 코드는 next.js, TypeScript, Swift WkWebView를 사용중인 코드인데 로컬 환경(run dev)에서는 아주 console.log("重要！", allClients);가 잘 표시 되는데 서버 환경에선 공백으로 표시돼
그리고 이건 swr쓰지만 swr를 쓰지 않는 같은 인쇄코드는 아주 문제 없이 작동되고 있어 이유가 뭘까?
```
