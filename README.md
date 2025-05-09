npm install react-use

import React, { useState } from "react";

interface BluetoothDevice {
  name?: string;
  id: string;
}

const BluetoothDeviceList: React.FC = () => {
  const [devices, setDevices] = useState<BluetoothDevice[]>([]);
  const [error, setError] = useState<string | null>(null);

  const requestBluetoothDevices = async () => {
    setError(null);

    try {
      const device = await navigator.bluetooth.requestDevice({
        acceptAllDevices: true,
        optionalServices: ["battery_service"], // 원하는 서비스 UUID
      });

      if (device.name) {
        setDevices((prev) => [
          ...prev,
          { name: device.name, id: device.id },
        ]);
      } else {
        console.warn("이름이 없는 기기는 무시됩니다.");
      }
    } catch (err) {
      setError((err as Error).message);
    }
  };

  return (
    <div>
      <h1>Bluetooth Device Scanner</h1>
      <button onClick={requestBluetoothDevices}>Scan for Devices</button>
      
      {error && <p style={{ color: "red" }}>{error}</p>}
      
      {devices.length > 0 ? (
        <ul>
          {devices.map((device) => (
            <li key={device.id}>
              {device.name} (ID: {device.id})
            </li>
          ))}
        </ul>
      ) : (
        <p>No devices found</p>
      )}
    </div>
  );
};

export default BluetoothDeviceList;


ㅅㅔㄱ

import React, { useState } from "react";

interface BluetoothDevice {
  name?: string;
  id: string;
}

const BluetoothDeviceList: React.FC = () => {
  const [devices, setDevices] = useState<BluetoothDevice[]>([]);
  const [error, setError] = useState<string | null>(null);

  const requestBluetoothDevices = async () => {
    setError(null);

    try {
      const device = await navigator.bluetooth.requestDevice({
        acceptAllDevices: true,
        optionalServices: ["battery_service"],
      });

      if (device.name) {
        const newDevice = { name: device.name, id: device.id };
        setDevices((prev) => [...prev, newDevice]);

        // 리스트를 콘솔에 출력
        console.log("Detected Devices:", [...devices, newDevice]);
      } else {
        console.warn("이름이 없는 기기는 무시됩니다.");
      }
    } catch (err) {
      setError((err as Error).message);
      console.error(err);
    }
  };

  return (
    <>
      <button onClick={requestBluetoothDevices}>Scan for Devices</button>
    </>
  );
};

export default BluetoothDeviceList;