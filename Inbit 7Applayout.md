import './globals.css';
import Script from 'next/script';

export const metadata = {
  title: 'Inbit.lol | Launch board for indie apps and startup tools',
  description:
    'Win the #1 launch spotlight and get targeted traffic, real first users and genuine feedback for your indie app, startup tool or digital product.',
};

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Script src="https://checkout.razorpay.com/v1/checkout.js" strategy="afterInteractive" />
      </body>
    </html>
  );
}