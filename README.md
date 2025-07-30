const groupedItems = clients.filter(
  item =>
    item.created_date + 'T' + item.created_time === updatedTime &&
    item.clients_name === selectedClient[0].clients_name
);

// ❌ 기존: allClients 사용
// const groupedItems = allClients.filter(...

// ✅ 수정: clients 사용 (SWR)
const groupedItems = clients.filter(
  item =>
    item.created_date + 'T' + item.created_time === updatedTime &&
    item.clients_name === selectedClient[0].clients_name
);