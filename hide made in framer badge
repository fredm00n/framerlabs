import type { ComponentType } from "react"

export function withoutFramerBadge(Component): ComponentType {
    return (props) => {
        return (
            <>
                <style>
                    {`
                        #__framer-badge-container { display: none !important; }
                    `}
                </style>

                <Component {...props} />
            </>
        )
    }
}
